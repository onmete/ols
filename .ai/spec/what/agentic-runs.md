# Agentic Runs

Multi-phase AI workflows that diagnose and remediate cluster issues. An alert fires, the system analyzes it, proposes remediation options, and — with human approval — executes and verifies the fix.

## End-to-End Flow

### Phase 1: Trigger

An external event source creates an `AgenticRun` CR to initiate a workflow. Any authorized adapter, controller, CLI, or API client can create AgenticRuns — the operator reconciles them regardless of origin. Adapters are create-only — they never update or delete AgenticRuns after creation.

**Example — alerts-adapter (AlertManager events):**

1. The alerts-adapter polls OpenShift AlertManager for firing alerts on a configurable interval.
2. For each firing alert, the adapter computes a fingerprint (8-char prefix) and checks for an existing AgenticRun CR with a deterministic name derived from the fingerprint.
3. If no matching AgenticRun exists and the cooldown window has elapsed since the last AgenticRun for that fingerprint, the adapter creates a new `AgenticRun` CR in the alert's namespace with the alert metadata and a templated remediation request.

**Example — event-adapter (team-harness prototype; Jira + GitHub domains):**

4. The event adapter uses one image with a separate Deployment + ConfigMap per domain (`source: jira` or `source: github`). See `lightspeed-team-harness/.ai/spec/what/event-adapter.md`.
5. The Jira domain polls for issues in New and creates batch triage AgenticRuns (analysis + human-approved execution).
6. The GitHub PR-review domain polls allowlisted repos and creates one AgenticRun per `repo + pull + headSha` after CI is terminal (all checks except Tide).

**Analysis-only writeback:** Some domains (e.g. GitHub PR review) perform external side effects during analysis (such as posting a Pull Request Review with event `COMMENT`) and return `actionRequired=false`, so the run terminates in `NoActionRequired` without an execution phase. This intentionally bypasses the propose → approve → execute gate for that domain and must be documented on the adapter; it does not change the CRD.

### Phase 2: Analysis

5. The agentic-operator detects the new AgenticRun CR and adds a finalizer.
6. The operator checks the cluster-scoped `ApprovalPolicy` (singleton named "cluster") for the analysis approval gate.
7. If approval is required, the operator waits for an `AgenticRunApproval` CR granting analysis. If automatic, it proceeds immediately.
8. The operator creates an input ConfigMap with the analysis query, output schema, context, and a pre-filled Result CR template. It then provisions a sandbox pod (bare-pod or sandbox-claim mode) with the ConfigMap mounted at `/input/`. [OLS-3066]
9. The sandbox pod runs the agent autonomously (batch execution — no HTTP). The agent executes using the configured LLM provider (Anthropic, Gemini, or OpenAI) and produces structured remediation options. Each option contains a concrete remediation script (ordered bash commands using kubectl/oc) and RBAC requirements derived from those commands. The analysis prompt instructs the agent to inspect cluster state with kubectl/oc before diagnosing, and to derive RBAC by tracing every command in its script. [OLS-3066]
10. The sandbox creates the `AnalysisResult` CR via `oc create` + `oc patch --subresource=status`, merging the agent output into the operator-provided template. The operator watches for this CR via `Owns()` and is automatically enqueued when it appears. [OLS-3066]
11. The operator reads the `AnalysisResult` CR and updates the AgenticRun conditions accordingly.
12. The analysis output includes an `actionRequired` boolean and a top-level `Diagnosis` (summary, rootCause). When `actionRequired` is false, the `Options` array may be empty (`minItems: 0`); the top-level `Diagnosis` captures the agent's explanation of why no remediation is needed.
13. When the operator stores an `AnalysisResult` with `actionRequired=false`, it sets the `Analyzed` condition to `True` with reason `NoActionRequired`. The AgenticRun auto-transitions to the `NoActionRequired` terminal phase, bypassing Proposed/Approval/Execution/Verification entirely.

### Phase 3: Approval

14. The agentic-console displays the AgenticRun in "Proposed" phase with the analysis results.
15. A human reviewer selects a remediation option, sets a max retry count, and creates an `AgenticRunApproval` CR for execution. **Only cluster-admin users may approve runs** — see `agentic-security.md` for authorization rules and enforcement.
16. The reviewer can optionally provide revision feedback via `spec.revisionFeedback` on the AgenticRun. Revision feedback is also supported from the `NoActionRequired` terminal phase — patching `spec.revisionFeedback` resets conditions and re-runs analysis, same as the re-analysis pattern from other phases.

### Phase 4: Execution

17. The operator materializes RBAC (ServiceAccount, Role, RoleBinding) scoped to the approved option's requirements.
18. The operator creates an input ConfigMap with the execution query (approved option, RBAC context) and provisions a sandbox pod. The execution prompt instructs the agent to follow the concrete bash script exactly and to dry-run mutation commands with `--dry-run=server` before executing. [OLS-3066]
19. The sandbox agent executes the remediation actions by running the approved bash commands in order.
20. The sandbox creates the `ExecutionResult` CR via `oc`, and the operator processes it upon watch notification. [OLS-3066]

### Phase 5: Verification

21. If verification is configured, the operator checks the approval gate for verification.
22. The operator calls the sandbox with a verification request, passing the execution result.
23. If verification fails, the operator retries up to max attempts, including previous attempt results as context.
24. On success, the operator stores the result in a `VerificationResult` CR and the AgenticRun moves to Completed.
25. On exhausted retries, the AgenticRun may escalate.

### Phase 6: Escalation

26. If verification fails after all retries, the operator checks the approval gate for escalation.
27. The operator calls the sandbox with an escalation request to generate a human-readable summary.
28. The result is stored in an `EscalationResult` CR and the AgenticRun moves to Escalated.

### Cleanup

29. On terminal phases (Completed, Failed, Denied, Escalated, NoActionRequired) or AgenticRun deletion, the operator deletes materialized RBAC, releases sandbox pods/claims, and removes the finalizer.

## Integration Contracts

### CRDs — `agentic.openshift.io/v1alpha1`

| CRD | Scope | Owner | Purpose |
|---|---|---|---|
| `AgenticRun` | Namespace | external adapters/clients (creates), operator (reconciles) | Workflow state machine. Immutable spec, mutable revisionFeedback, status conditions. |
| `AgenticRunApproval` | Namespace | console (creates) | Approval decisions per stage, option selection, max attempts override. Owned by AgenticRun. |
| `ApprovalPolicy` | Cluster (singleton "cluster") | admin (creates) | Automatic/Manual gates per stage, max attempts, max concurrent runs. |
| `Agent` | Cluster | admin (creates) | LLM provider selection and model name. |
| `LLMProvider` | Cluster | admin (creates) | Provider type, credentials secret, URL, region/project. |
| `AnalysisResult` | Namespace | sandbox (creates via `oc`), operator (reads) [OLS-3066] | Immutable analysis output. Owned by AgenticRun. |
| `ExecutionResult` | Namespace | sandbox (creates via `oc`), operator (reads) [OLS-3066] | Immutable execution output. Owned by AgenticRun. |
| `VerificationResult` | Namespace | sandbox (creates via `oc`), operator (reads) [OLS-3066] | Immutable verification output. Owned by AgenticRun. |
| `EscalationResult` | Namespace | sandbox (creates via `oc`), operator (reads) [OLS-3066] | Immutable escalation output. Owned by AgenticRun. |

### [OLS-3066] Batch Sandbox I/O (replaces HTTP)

The operator and sandbox communicate via Kubernetes objects, not HTTP:

| Direction | Mechanism | Content |
|---|---|---|
| Operator → Sandbox (input) | ConfigMap volume mount at `/input/` | `query` (rendered prompt), `output-schema` (JSON schema), `context` (targetNamespaces, previousAttempts, approvedOption, executionResult), `result-template` (pre-filled Result CR) |
| Sandbox → Operator (output) | Result CR created via `oc create` + `oc patch --subresource=status` | Same Result CR status fields as before (options, diagnosis, actionsTaken, checks, conditions, failureReason) |
| Sandbox → Operator (errors) | `/dev/termination-log` (sandbox failures) or Result CR with `failureReason` (agent failures) | Error message string |

Context envelope in the `context` ConfigMap key varies by phase:
- Analysis: target namespaces
- Execution: approved option (diagnosis, actions, RBAC), target namespaces
- Verification: execution result, previous attempts, attempt metadata
- Escalation: full workflow history

### Shared Data Formats

- **Alert fingerprint**: 8-char prefix for deterministic AgenticRun naming and deduplication
- **AnalysisResult schema**: includes `actionRequired` (bool) and a top-level `Diagnosis` (summary, rootCause). When `actionRequired` is false, `Options` may be empty. Each `RemediationOption` contains diagnosis, remediation plan (`plan` field), RBAC requirements, verification plan. The `RemediationPlan` struct holds description, actions, and reversibility. Each action includes `command` (exact bash command, required, 1-4096 chars), `type` (phase category: pre-check, mutation, wait, post-check), and `description`. RBAC requirements are derived from the script commands, with `get`/`list`/`watch` as minimum read verbs for every resource.
- **Phase derivation**: from status.conditions with precedence EmergencyStopped > Escalated > Denied > Verified > Executed > Analyzed (with `NoActionRequired` reason → `NoActionRequired` phase, otherwise → Proposed)
- **LLM config env vars**: `LIGHTSPEED_PROVIDER`, `LIGHTSPEED_MODEL`, `LIGHTSPEED_PROVIDER_URL`, and region/project/api-version variants

## Repo Ownership

| Repo | Owns |
|---|---|
| **lightspeed-agentic-alerts-adapter** | Alert polling, fingerprint-based dedup, cooldown enforcement, AgenticRun CR creation (create-only) |
| **lightspeed-agentic-operator** | AgenticRun reconciliation, approval gate enforcement, sandbox provisioning (ConfigMap input + pod creation), RBAC materialization, Result CR processing (reads CRs created by sandbox), phase derivation, finalizer cleanup [OLS-3066] |
| **lightspeed-agentic-sandbox** | Batch agent execution (reads `/input/`, runs LLM, creates Result CR via `oc`), LLM provider abstraction (DeepAgents/Anthropic, Gemini, OpenAI adapters), structured output handling, tool execution, event logging [OLS-3066] |
| **lightspeed-agentic-console** | AgenticRun list/detail UI, phase display (mirrors operator's phase derivation), approval decision UI, option selection, revision feedback, escalation display |

## Planned Changes

| Ticket | Summary |
|---|---|
| OLS-3066 | Decouple reconcile latency: batch sandbox model, ConfigMap input, Result CR output via `oc`, watch-driven async, per-step timeout, ≤30s reconcile SLO. Subsumes OLS-2913 step-conditions. |
| OLS-2894 | Per-run approval overrides and namespace-scoped `ApprovalPolicy` |
| OLS-2957 | Sandbox template management UX and CRD ergonomics |
| ~~OLS-3038~~ | ~~TLS verification and network policy for agent traffic~~ No longer applicable — sandbox pods have no HTTP server (OLS-3066) |
| OLS-3033 | Operator-passed `allowedTools` and `llm` aligned with `ProviderQueryOptions` |
| ~~OLS-3268~~ | ~~Analysis can signal `actionRequired=false` to auto-complete with `NoActionRequired` phase~~ [DONE: OLS-3268] |
| ~~OLS-3295~~ | ~~Rename `Proposal` → `AgenticRun`, `ProposalApproval` → `AgenticRunApproval`, `ProposalResult` → `RemediationPlan` across CRDs, API, CLI, console, and docs~~ [DONE: OLS-3295] |
| OLS-3441 | Script-grounded RBAC: analysis produces concrete bash scripts and derives RBAC from commands; execution dry-runs mutations before applying |
| OLS-3657 | Event adapter: Jira-triggered AgenticRuns for automated bug triage (prototype in lightspeed-team-harness) |

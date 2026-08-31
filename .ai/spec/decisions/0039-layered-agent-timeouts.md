# 0037: Layered Agent and Sandbox Timeouts

**Status:** Accepted [PLANNED: OLS-3743]
**Applies to:** lightspeed-agentic-operator, lightspeed-agentic-sandbox

## Context

`Agent.spec.timeouts` declares analysis, execution, verification, and chat timeout fields, but the agentic operator does not use them. `Agent.spec.maxTurns` is also not passed to the sandbox. The legacy synchronous implementation instead has independent five-minute defaults for sandbox readiness, the operator HTTP client, and the sandbox agent request. The OLS-3743 HTTP wiring proposal is obsolete because OLS-3066 replaces operator-to-sandbox HTTP with a batch pod and Result CR model.

The batch sandbox currently applies one internal wall-clock timer around the complete agent invocation. This is not an individual LLM request timeout: it covers model calls, tool and skill execution, MCP calls, and processing across turns. The operator also needs a hard pod deadline because SDK cancellation is cooperative and some providers may remain blocked. A single enforcement mechanism therefore cannot provide both graceful failure reporting and guaranteed resource cleanup.

## Decision

### One user-facing execution budget, two enforcement layers

`Agent.spec.timeouts` is the only user-facing timeout configuration. Each field defines the agent execution budget for one workflow step:

| Field | Default | Validation |
|---|---:|---:|
| `analysisSeconds` | 600 seconds | 1–3600 seconds |
| `executionSeconds` | 600 seconds | 1–3600 seconds |
| `verificationSeconds` | 1800 seconds | 1–3600 seconds |
| `escalationSeconds` | 600 seconds | 1–3600 seconds |

Fields remain optional. The operator resolves omitted values to the defaults above; the API server does not materialize defaults into the CR.

The operator passes the effective step value to the sandbox as `LIGHTSPEED_AGENT_TIMEOUT_SECONDS`. This directly replaces `LIGHTSPEED_TIMEOUT_MS`; no compatibility alias is required. The sandbox requires a positive integer and treats a missing, zero, negative, or malformed value as a sandbox configuration failure. The operator is the sole owner of defaults so Go and Python defaults cannot diverge.

The sandbox applies the value as a cooperative wall-clock limit around the complete agent invocation. If it expires, the sandbox publishes the step Result CR with `failureReason`, a `Completed=True` condition whose reason is `AgentTimeout`, and exits successfully as a sandbox process.

### Fixed operator lifecycle margins

The operator enforces two additional, non-configurable deadlines:

1. **Startup deadline:** five minutes from Pod creation in bare-pod mode or SandboxClaim creation in sandbox-claim mode. It expires when the main sandbox container has not started.
2. **Running deadline:** main container `startedAt` plus the effective agent timeout plus one minute. The additional minute covers sandbox initialization, cancellation handling, and Result CR publication.

On deadline expiry, the operator marks the step terminal, releases or deletes its Pod/SandboxClaim, and deletes the input ConfigMap. The operator derives deadlines from Kubernetes timestamps so reconciliation after restart preserves the original budget. While a step is active, it requeues after the smaller of 30 seconds or the time remaining to the applicable deadline.

The lifecycle margins are implementation policy and are not exposed through `AgenticOLSConfig` or `Agent` in OLS-3743.

### Status and retry behavior

Timeout sources remain distinguishable:

| Source | Step condition reason |
|---|---|
| Container did not start before the startup deadline | `SandboxStartupTimeout` |
| Sandbox reported cooperative agent timeout | `AgentTimeout` |
| Operator running deadline expired | `SandboxTimeout` |

A generic agent failure remains `AgentFailed`; a pod or process failure remains `SandboxFailed`. Messages include the step and effective duration.

Timeouts never trigger automatic retries, including verification timeouts. Repeating the same work with the same budget is unlikely to succeed, and retrying execution can duplicate mutations. A user may revise or recreate the run after changing the selected Agent configuration. If a Result CR races with operator timeout cleanup, the already-terminal step condition wins and the late result is ignored.

### Agent API cleanup and turn limit

`chatSeconds` is removed from `Agent.spec.timeouts`. The supported providers do not expose one consistent per-chat-turn timeout, so retaining it would promise behavior the product cannot enforce.

`escalationSeconds` is added because escalation is a distinct workflow step and must not implicitly inherit the analysis timeout.

`Agent.spec.maxTurns` remains optional with its existing 1–500 validation and a default of 200 resolved by the operator. The operator passes the effective value as `LIGHTSPEED_AGENT_MAX_TURNS`. As with the timeout environment variable, the sandbox requires a valid value in the API range and treats missing or malformed configuration as a sandbox failure. The sandbox forwards it through `ProviderQueryOptions.max_turns` to each provider's corresponding SDK limit:

- DeepAgents/LangGraph: recursion limit
- Gemini ADK: maximum LLM calls
- OpenAI Agents: maximum turns

These SDK mechanisms are not identical, but they provide the common product contract of an upper bound on provider iteration. The product does not rely on differing SDK defaults.

### Implementation boundary

This decision targets the OLS-3066 batch architecture and supersedes OLS-3066's original single ten-minute deadline measured from Pod creation. The obsolete synchronous HTTP client and request `timeout_ms` path are not wired or extended. OLS-3743 implementation must update the agentic operator and batch sandbox contracts together.

## Alternatives Considered

- **Expose separate agent and sandbox timeout values for every step** — rejected because users would have to understand internal enforcement layers, invalid combinations could kill the pod before it reports failure, and most configurations would repeat an arbitrary grace interval.
- **Use one total deadline from sandbox creation** — rejected because scheduling and image pulls would consume the agent's configured execution budget.
- **Rely only on the sandbox timer** — rejected because cooperative cancellation cannot guarantee pod cleanup when an SDK or process is blocked.
- **Rely only on the operator deadline** — rejected because forced deletion loses graceful timeout reporting and Result CR publication.
- **Put lifecycle margins in `AgenticOLSConfig`** — deferred because there is no current requirement for administrators to tune infrastructure margins.
- **Remove `maxTurns` and use SDK defaults** — rejected because provider defaults and semantics differ and may change with dependency upgrades.

## Consequences

- Administrators configure one execution budget per workflow step instead of coordinating two timeout values.
- The sandbox normally reports an actionable agent timeout before the operator's hard deadline.
- Stuck sandbox processes remain bounded and are cleaned up after a deterministic deadline.
- Verification defaults to 30 minutes; all other steps default to 10 minutes.
- The `Agent` v1alpha1 API removes an unwired field and adds an escalation timeout field; generated CRDs, examples, and documentation must change.
- Operator tests must cover default and configured resolution, selected-agent and approval-override behavior, environment injection, timestamp-derived deadlines, restart safety, deadline-aware requeue, cleanup races, status reasons, and no timeout retries.
- Sandbox tests must cover timeout and max-turn environment parsing, provider option forwarding, cooperative timeout Result CR status, and malformed configuration failures.

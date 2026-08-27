# MCP Tool RBAC Resolution

How the agentic system derives least-privilege Kubernetes RBAC for remediation steps that are **MCP tool calls** (rather than `oc`/`kubectl` commands), so the operator can materialize the correct permissions onto a per-step ServiceAccount before execution. Extends the script-grounded RBAC model (`agentic-runs.md` Phase 4, decision [0033](../decisions/0033-script-grounded-rbac.md)) to the tool-call case. Motivated by [OLS-3680](https://redhat.atlassian.net/browse/OLS-3680). Design rationale: decision [0038](../decisions/0038-mcp-tool-rbac-resolution.md).

## Problem

Execution steps run as a dedicated per-step ServiceAccount whose bearer token is passed through to the API server; the OpenShift MCP server holds no RBAC of its own and authorizes as that token (see `agentic-security.md` and `lightspeed-operator/.ai/spec/what/ocpmcp.md`). For `oc`/`kubectl` steps, the operator derives least-privilege RBAC by tracing the concrete commands. For MCP tool calls there is no command to trace, and the RBAC target is often invisible in the tool arguments:

- **Subresources** implied by the tool identity, not the arguments — `pods_exec` → `create pods/exec`, `pods_log` → `get pods/log`, `nodes_log`/`nodes_stats_summary` → `nodes/proxy`, `resources_scale` → `*/scale`.
- **Generic pass-throughs** whose group/resource come from `apiVersion`/`kind` arguments — `resources_get`/`resources_list`/`resources_delete`.
- **Manifest-embedded GVK** — `resources_create_or_update` carries the target inside a free-form manifest argument.
- **Unbounded effect** — `helm_install`/`helm_uninstall` apply whatever the chart contains; RBAC is not a function of the arguments.

The enforcement boundary (token passthrough) is unaffected. The gap is **derivation and pre-flight validation**.

## Behavioral Rules

### Resolution order (per tool-call step)

1. **`_meta` contract (primary).** The operator resolves a tool's required RBAC from `tool._meta["openshift.io/rbac"]` returned by the MCP server's `tools/list`. The contract and its schema are defined by the OLS-3680 RFE to the OpenShift MCP server team.
2. **oc-IR derivation (fallback).** When a tool has no trusted `_meta` RBAC, the analysis agent MUST express the step's effect as equivalent `oc`/`kubectl` command intermediate representation, and RBAC is derived from it via the script-grounded pipeline (decision 0033). This is why the common ocp-mcp core tools remain resolvable even before the server adopts `_meta`.
3. **Fail-closed (terminal).** A step resolvable by neither path MUST cause the containing remediation **option** to be rejected — not the whole analysis. If every option is rejected, the run terminates in `Escalated` with a diagnosis naming the unresolvable tool(s) and, where applicable, which declaration would resolve it.

### `_meta` trust

4. **Operator-managed servers only.** `_meta` RBAC is honored **only** from MCP servers on the operator-managed allowlist (the shipped `openshift-mcp-server`). `_meta` advertised by bring-your-own / third-party MCP servers is untrusted and MUST be ignored — those tools resolve via oc-IR (rule 2) or fail closed (rule 3).
5. **Derivation input, not enforcement.** `_meta` and oc-IR are inputs to RBAC materialization only. They never widen or replace the enforcement boundary, which remains the API server acting on the per-step SA's passed-through token. A too-narrow declaration therefore yields an honest 403 at execution; the risk the trust rule guards against is an over-broad or dishonest declaration.

### Declaration states

6. **Four distinct states.** The operator MUST distinguish, for each tool:
   - `noRbac: true` — authoritatively needs no cluster RBAC (e.g. local-kubeconfig tools) → passes validation, grants nothing.
   - a resolvable form (`rules` / `deriveFromArgs` / `deriveFromManifest`) → resolve and grant.
   - `unbounded: true` — needs RBAC but cannot be bounded (e.g. Helm) → fail closed.
   - key absent or empty → undeclared → fall back to oc-IR, else fail closed.
7. **Empty is not "none".** An absent or empty `_meta` RBAC declaration MUST NOT be treated as "needs no RBAC". Only an explicit `noRbac: true` means that. This prevents a server that has not adopted the contract from silently running with no grant.
8. **Mutation/`noRbac` invariant.** A tool that mutates the Kubernetes API — write verbs in its declaration, or `annotations.destructiveHint: true` / `readOnlyHint: false` — MUST NOT declare `noRbac: true`. The operator MUST reject this contradiction as a mis-declaration.

### Resolution and materialization

9. **Argument-scoped resolution.** For `deriveFromArgs`/`deriveFromManifest` forms and for `rules` with argument references, the operator MUST resolve the rule against the **actual call arguments**, mapping `kind → resource` via the cluster's discovery/RESTMapper, and scope the materialized rule to the specific namespace/object where the contract provides `namespaceFrom`/`resourceNamesFrom`.
10. **`resourceNames` limitation.** The operator MUST NOT rely on `resourceNames` to scope `list`/`watch` — Kubernetes RBAC cannot restrict those verbs to named objects. Read-listing tools resolve to namespace-wide read.
11. **Deny ceiling.** Before materialization, the operator MUST intersect the resolved rules with a hard deny ceiling: never grant any verb on `core/v1` `Secret` or on any `rbac.authorization.k8s.io` resource, regardless of what `_meta` or oc-IR resolved. This mirrors the ocp-mcp TOML denies. A step that genuinely requires a denied resource fails closed (rule 3). Matching against the ceiling MUST follow defined semantics so a grant cannot slip past it: the core API group is normalized as `""` (`core` ≡ `""`); resources are compared as RESTMapper plurals and a denied resource also denies its subresources (`secrets` ⇒ `secrets/<sub>`); a `*` in `apiGroups`/`resources`/`verbs` is expanded-or-rejected so a wildcard cannot bypass a denied `(group, resource)`; and `resourceNames` never exempts a rule from denial. The normative rules and conformance table live in `lightspeed-agentic-operator/.ai/spec/what/sandbox-execution.md` rule 21d.
12. **Union and provenance.** The operator MUST union the resolved rules across all steps of the approved option, merging only identical `(apiGroups, resources, resourceNames)` tuples by unioning their verbs, and MUST NOT collapse a `resourceNames`-scoped rule into an unscoped one. Each materialized rule carries provenance identifying its source (`meta:<server>/<tool>` or `derived:oc:<command>`).
13. **Split by scope.** Namespaced rules materialize as Role/RoleBinding; cluster-scoped rules as ClusterRole/ClusterRoleBinding. Both bind to the per-step execution SA (`agentic-security.md` rule 7), never the shared `lightspeed-agent` SA.

## Integration Contracts

### `_meta["openshift.io/rbac"]` (MCP `tools/list`)

The MCP server publishes, per tool, a versioned RBAC document. Forms: `rules` (static, optionally argument-scoped), `deriveFromArgs` (target from named arguments), `deriveFromManifest` (target from a manifest argument), `unbounded: true`, `noRbac: true`. Full schema and examples are specified in the OLS-3680 RFE to the OpenShift MCP server team. This is an **external dependency** — the OpenShift MCP server must implement it (RFE pending); until then, tools resolve via oc-IR or fail closed.

### RemediationOption RBAC

The RBAC requirements attached to each `RemediationOption` (`agentic-runs.md` — AnalysisResult schema) are the union computed by rule 12, provenance-tagged. The operator materializes them in Phase 4 (`agentic-runs.md` rule 17) before provisioning the execution pod.

## Repo Ownership

| Repo | Owns |
|---|---|
| **lightspeed-operator** (ocp-mcp) | Publishing `_meta["openshift.io/rbac"]` on the shipped `openshift-mcp-server` (RFE target); the deny-list TOML the ceiling mirrors |
| **lightspeed-agentic-operator** | `_meta` fetch + allowlist trust check, resolution against call arguments (RESTMapper), deny-ceiling intersection, union/provenance, RBAC materialization onto the per-step SA, option-level fail-closed and `Escalated` transition |
| **lightspeed-agentic-sandbox** | Analysis-time oc-IR fallback (expressing MCP tool steps as equivalent `oc` commands when `_meta` is unavailable) |

## Child Spec Updates Required

These child specs describe behavior this file extends. Each MUST be updated (separate PRs in their repos):

| Repo | Spec File | Update |
|---|---|---|
| lightspeed-operator | `what/ocpmcp.md` | Add rule: shipped server publishes per-tool `_meta["openshift.io/rbac"]` (RFE); note deny-list continuity with the consumer-side ceiling. |
| lightspeed-agentic-operator | `what/sandbox-execution.md` rule 11 | Analysis derives MCP-tool RBAC from trusted `_meta` first, oc-IR fallback otherwise. |
| lightspeed-agentic-operator | `what/sandbox-execution.md` rule 21 | Materialization consumes provenance-tagged rules from `_meta`/oc-IR; apply deny ceiling; option-level fail-closed. |
| lightspeed-agentic-sandbox | `what/configuration.md` | Note oc-IR expression of MCP tool steps for RBAC derivation when `_meta` is absent. |

## Constraints

- `_meta` is untrusted per the MCP specification; trust is granted only to operator-managed servers and is always backstopped by the deny ceiling.
- The `metrics` toolset (Thanos/Alertmanager, per `ocpmcp.md` rule 17) may authorize via monitoring routes / aggregated APIs rather than resource CRUD; expressing that RBAC (e.g. binding `cluster-monitoring-view`) is an open item raised in the RFE and not yet modeled here.
- The design cannot compute least-privilege RBAC for genuinely unbounded tools (Helm); those fail closed unless resolved by other means.

## Planned Changes

| Ticket | Summary |
|---|---|
| [PLANNED: OLS-3680] | MCP tool RBAC resolution: `_meta`-published contract (operator-managed servers) → oc-IR fallback → fail-closed, with a hard deny ceiling. RFE to the OpenShift MCP server team for `_meta["openshift.io/rbac"]`. |
| [PLANNED] | Non-resource / aggregated-API RBAC form for the `metrics` toolset (Thanos/Alertmanager), pending alignment in the RFE. |

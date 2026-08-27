# 0038: RBAC Resolution for MCP Tool Calls

**Status:** Accepted
**Applies to:** lightspeed-agentic-operator, lightspeed-agentic-sandbox, lightspeed-operator (ocp-mcp)
**Related:** [0033](0033-script-grounded-rbac.md) (script-grounded RBAC — extended, not replaced), [0013](0013-mcp-for-tool-integration.md) (MCP for tool integration)

## Context

Agentic execution derives least-privilege RBAC for a remediation by tracing the concrete `oc`/`kubectl` commands the analysis agent produced (decision 0033). Each step runs as a dedicated per-step ServiceAccount whose token is passed through to the API server; the OpenShift MCP server holds no RBAC of its own and authorizes as the caller's token (`lightspeed-operator/.ai/spec/what/ocpmcp.md`).

When a remediation step is an **MCP tool call** rather than a bash command, there is no script to trace. The RBAC target is frequently invisible in the tool arguments: subresources (`pods/exec`, `pods/log`, `nodes/proxy`, `*/scale`) implied by the tool's identity, generic pass-throughs whose group/resource come from `apiVersion`/`kind` arguments (`resources_delete`), a GVK embedded inside a free-form manifest argument (`resources_create_or_update`), or tools whose effect is unbounded (`helm_install` applies arbitrary chart contents). `ToolAnnotations` (`readOnlyHint`/`destructiveHint`) distinguish read from mutate but do not identify the resource or verb, and the MCP specification states annotations MUST NOT be the sole basis for security decisions.

Without a reliable RBAC source, execution either over-grants (violating least-privilege) or 403s at runtime. The enforcement boundary (per-step SA token passthrough) is unaffected — the gap is **derivation and pre-flight validation**.

## Decision

Resolve the RBAC for each MCP tool-call step in a fixed precedence, and materialize the result onto the per-step ServiceAccount before the step's pod runs:

1. **Server-published `_meta` contract (primary).** The MCP server advertises per-tool required RBAC under `tool._meta["openshift.io/rbac"]` in `tools/list`, using static `rules`, argument-derived (`deriveFromArgs`), manifest-derived (`deriveFromManifest`), `unbounded: true`, or `noRbac: true` forms. This contract is the subject of an RFE to the OpenShift MCP server team (OLS-3680). It is honored **only from operator-managed, allowlisted MCP servers**; `_meta` from bring-your-own MCP servers is untrusted and ignored.
2. **oc-IR derivation (fallback).** For a tool whose `_meta` is absent/untrusted, the analysis agent expresses the step's effect as equivalent `oc` command intermediate representation and the existing script-grounded pipeline (0033) derives RBAC from it.
3. **Fail-closed (terminal).** A step resolvable by neither path — an opaque tool (`unbounded: true`, or not oc-expressible) on a server whose `_meta` cannot be trusted — causes the remediation option to be rejected at option level; the run terminates in `Escalated` with a diagnosis naming the unresolvable tool.

Whatever resolves is intersected with a **hard deny ceiling** (never grant `core/v1` `Secret` or any `rbac.authorization.k8s.io` resource) before materialization, mirroring the ocp-mcp TOML denies. `_meta` and oc-IR are derivation inputs only; enforcement remains the API server acting on the passed-through token.

## Alternatives Considered

- **Pure argument-based derivation / prediction** — rejected. Cannot see subresources, manifest-embedded GVKs, or unbounded effects; every consumer re-reverse-engineers each tool and drifts as the server changes. This is the band-aid the `_meta` contract replaces.
- **Out-of-band CRD of tool→RBAC maintained by OLS** (`AgenticMCPToolRBACList`) — rejected as the primary mechanism. Places the declaration in the wrong owner (the tool author knows the answer), drifts from the server, and scales poorly. The schema work was folded into the `_meta` contract instead.
- **Trust `_meta` from any MCP server** — rejected. `_meta` is untrusted per the MCP spec; an arbitrary server could over-declare to obtain a broad grant for its SA. Trust is limited to operator-managed servers, backstopped by the deny ceiling.
- **No deny ceiling (rely solely on the cluster-admin approval gate)** — rejected. A single mis-declaration or dishonest server should not be able to reach Secret/RBAC data; defense-in-depth caps the worst case regardless of what any source declares.
- **MCP Authorization (OAuth 2.1)** — not applicable. It governs client access to the MCP server, not the downstream Kubernetes RBAC a tool call requires.

## Consequences

- Least-privilege RBAC is derivable for MCP tool calls, including subresource and manifest-embedded targets, without hard-coding tool knowledge in OLS.
- The authoritative RBAC declaration lives with the tool author and travels with the server, discovered over the protocol already in use — pending the RFE landing.
- The design has an external dependency: until the OpenShift MCP server populates `_meta`, tools resolve via the oc-IR fallback or fail closed.
- Opaque tools (Helm) and untrusted BYO MCP servers fail closed rather than over-grant, which may block some remediation options until an operator-managed server declares them.
- A hard deny ceiling guarantees Secret/RBAC access is never materialized onto an execution SA, independent of the RBAC source.
- Consumers must distinguish `noRbac: true` from an absent/empty declaration; empty is treated as undeclared (fail-closed / fallback), never as "needs nothing."

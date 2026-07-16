# OLS-1062 — OLS Chat CLI Plugin (oc-ols) Architecture Decisions

**Date:** 2026-07-16
**Status:** Accepted
**Ticket:** [OLS-1062](https://redhat.atlassian.net/browse/OLS-1062)

---

## Context

OpenShift admins and users who primarily interact via the CLI need a way to access OpenShift Lightspeed without using the web console. The agentic variant already has a CLI plugin (`oc-agentic` in `lightspeed-agentic-operator`) for managing AgenticRun CRDs. A similar plugin is needed for the core OLS chat use case.

A deep interview (9 rounds, 20% ambiguity score) and consensus planning (Planner/Architect/Critic) were conducted to establish requirements and resolve architectural decisions before implementation.

---

## Decision 1: Repository — Go binary in lightspeed-operator

**Decision:** The `oc-ols` CLI lives in the `lightspeed-operator` repository.

**Drivers:**
- The operator repo already has Go toolchain, CI, and manages OLS deployment
- Mirrors the pattern where `oc-agentic` lives in `lightspeed-agentic-operator`
- The CLI ships alongside the component that deploys the service it talks to

**Alternatives considered:**
- **lightspeed-service repo** — Rejected: service is Python, adding Go build toolchain adds complexity
- **New dedicated repo** — Rejected: more repos to manage, the operator is the natural home
- **Python CLI** — Rejected: inconsistent with `oc-agentic` pattern, harder to distribute as a kubectl plugin

---

## Decision 2: Architecture — REST API client, not K8s CRD client

**Decision:** The CLI uses `client-go` for kubeconfig/token/TLS reading only, and `net/http` for REST API calls to lightspeed-service. No `controller-runtime`, no CRD interactions.

**Drivers:**
- The OLS chat CLI sends HTTP requests to a REST API — fundamentally different from `oc-agentic` which manages K8s CRDs
- `controller-runtime` adds unnecessary dependency weight for an HTTP client
- `client-go` is still needed to read kubeconfig for bearer token extraction and TLS configuration

**Alternatives considered:**
- **Full controller-runtime like oc-agentic** — Rejected: overkill for an HTTP client, adds CRD scheme registration overhead
- **Shell script / curl wrapper** — Rejected (via contrarian challenge): lacks kubectl plugin discoverability, cross-platform distribution, and kubeconfig integration

---

## Decision 3: Service Discovery — Admin-created Route, user-configured endpoint

**Decision:** The operator does NOT auto-discover or auto-expose the service endpoint. The admin creates an OpenShift Route manually (per existing documentation), and the user configures the CLI with `oc ols config set-endpoint <URL>`. The endpoint is stored per kubeconfig context.

**Drivers:**
- The operator does not create Routes today — this is by design, not an oversight
- The OLSConfig CR (cluster-scoped) has no endpoint URL field in spec or status
- The console accesses the service via a ConsolePlugin proxy, which doesn't work for CLI tools
- Adding operator changes for service discovery would expand scope beyond OLS-1062

**Alternatives considered:**
- **Add endpoint to OLSConfig status** — Rejected: requires operator changes, out of scope
- **K8s API server proxy** — Rejected: requires service proxy RBAC grants
- **Console proxy reuse** — Rejected: browser-only mechanism
- **Environment variable only** — Rejected in favor of per-context storage for multi-cluster UX

---

## Decision 4: Command Structure — Default mode dispatching

**Decision:** The CLI supports `oc ols "question"` (defaults to `ask` mode), `oc ols ask "question"` (explicit), and `oc ols troubleshoot "question"` (explicit). Cobra resolves subcommands first; unmatched first args fall through to the root command as a query.

**Drivers:**
- `oc ols "question"` is the simplest UX for the most common case
- `--mode=troubleshoot` on every invocation is worse UX than `oc ols troubleshoot`
- Cobra's subcommand resolution handles both patterns naturally

---

## Decision 5: Conversation Persistence — Auto-persist with transparency

**Decision:** The CLI auto-saves `conversation_id` per kubeconfig context in `~/.config/oc-ols/contexts/<context-name>/`. Subsequent queries auto-continue the conversation. The CLI always prints "Continuing conversation \<id\>..." to stderr when continuing. `--new` starts fresh.

**Drivers:**
- Multi-turn conversations improve answer quality (service uses conversation history for context)
- Users must always be informed when continuing — hidden state changes are confusing
- Per-context storage ensures switching clusters doesn't leak conversation state

**Alternatives considered:**
- **Show-and-pass (manual --conversation-id)** — Rejected: too much friction for the common case
- **No continuity in v1** — Rejected: conversation context is valuable and the API already supports it

---

## Decision 6: Output Rendering — Terminal markdown via glamour

**Decision:** LLM responses are rendered through charmbracelet/glamour for terminal-friendly markdown (headings, code blocks, lists). `--output json` bypasses rendering for scripting. Non-TTY output (piped) auto-falls back to plain text.

**Drivers:**
- LLM responses contain markdown formatting that looks poor as raw text in a terminal
- glamour auto-detects terminal width and color support
- `--output json` provides the raw structured response for programmatic use

---

## Decision 7: Build & Distribution — Mirror agentic CLI pattern

**Decision:** GoReleaser with 4 targets (linux/darwin × amd64/arm64), GitHub Actions CI on push to main (path-filtered), rolling `latest` GitHub Release with tar.gz archives and checksums. No semver, no brew, no RPM/DEB.

**Drivers:**
- Identical pattern already proven with `oc-agentic`
- Cross-platform coverage matches the agentic CLI
- Rolling `latest` release provides stable download URLs

---

## Decision 8: Spec Documentation — Mirror agentic spec pattern with adapted headings

**Decision:** Create `lightspeed-operator/.ai/spec/how/cli.md` and `cli-distribution.md` following the agentic CLI specs' organizational pattern (entry point → module map → command tree → data flow → abstractions → cross-references) but with section headings adapted to the REST API client architecture (e.g., "Service client & streaming" instead of "Kubernetes client usage").

**Drivers:**
- User explicitly requested mirroring the agentic pattern (deep interview Round 3)
- Architect identified that force-fitting agentic headings onto a REST client would be misleading
- Adapted headings preserve navigability while accurately describing the architecture

---

## Consequences

**Positive:**
- Implementation agents get a precise blueprint with 25 acceptance criteria
- Cross-repo navigability preserved via consistent spec organizational patterns
- Users get a simple `oc ols "question"` interface with no setup beyond endpoint configuration

**Negative:**
- Endpoint configuration requires a manual setup step (admin Route + user config)
- Conversation persistence adds local state that users must manage (manual cleanup)

**Follow-ups:**
- OLS-1062 implementation: create the actual CLI code following the specs
- Update spec module maps with actual file names during implementation
- Consider adding endpoint auto-discovery in a future iteration if demand warrants operator changes

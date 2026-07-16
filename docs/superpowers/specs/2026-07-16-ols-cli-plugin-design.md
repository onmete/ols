# OLS Chat CLI Plugin (`oc-ols`)

Design for a kubectl/oc CLI plugin that provides command-line access to OpenShift Lightspeed chat, enabling administrators and users who prefer CLI workflows to interact with OLS directly from the terminal.

**Jira**: OLS-1062
**Specs**: `lightspeed-operator/.ai/spec/how/cli.md`, `lightspeed-operator/.ai/spec/how/cli-distribution.md`

## Scope

**In scope:**

- One-shot streaming chat queries via `oc ols "question"`
- Explicit mode selection: `oc ols ask` (default) and `oc ols troubleshoot`
- File attachments via `--file` flag (logs, configs, YAML)
- Conversation continuity: auto-persisted `conversation_id` per kubeconfig context
- Terminal markdown rendering (glamour) for readable LLM output
- Structured JSON output (`--output json`) for scripting
- Per-kubeconfig-context endpoint configuration
- Cross-platform binary distribution via GoReleaser (linux/darwin × amd64/arm64)

**Not in scope:**

- Conversation history management (list/get/delete) — backend cache is not a full history system
- Automatic service endpoint discovery — admin creates Route manually per existing docs
- Operator or OLSConfig changes for CLI support
- Interactive/REPL chat mode

## Design Decisions

### 1. Repository — `lightspeed-operator`

The CLI lives in `lightspeed-operator` rather than `lightspeed-service` (Python) or a new repo.

**Why:** The operator repo already has Go toolchain, GoReleaser, and GitHub Actions patterns from the agentic console/alerts adapter work. The operator manages the OLS deployment, so shipping the CLI alongside it follows the same pattern as `oc-agentic` living in `lightspeed-agentic-operator`.

**Alternatives considered:**
- `lightspeed-service` — rejected because the service is Python; adding a Go binary would introduce a second language toolchain
- New dedicated repo — rejected to avoid repo sprawl; the CLI is small enough to coexist
- Python CLI — rejected for consistency with `oc-agentic` and kubectl plugin conventions (single binary, no runtime dependencies)

### 2. Architecture — REST API Client, Not CRD Manager

Unlike `oc-agentic` (which uses controller-runtime for Kubernetes CRD CRUD), `oc-ols` is a pure HTTP client:

- **client-go** for kubeconfig parsing, bearer token extraction, and TLS configuration only
- **net/http** for REST API calls to lightspeed-service
- **No controller-runtime**, no CRD types, no API server interaction

**Why:** The OLS chat CLI sends HTTP requests to the lightspeed-service REST API (`POST /v1/streaming_query`). It does not create, read, or manage Kubernetes custom resources. The K8s dependency is limited to reading kubeconfig for authentication credentials and TLS settings.

### 3. Service Discovery — Admin-Created Route + User-Configured Endpoint

The CLI does **not** auto-discover the lightspeed-service endpoint.

**Service exposure model:**
1. Admin creates an OpenShift Route for `lightspeed-app-server` in `openshift-lightspeed` namespace (existing documented procedure)
2. User configures: `oc ols config set-endpoint <URL>`
3. Endpoint stored per kubeconfig context in `~/.config/oc-ols/contexts/<context-name>/endpoint`

**Resolution order:** `--endpoint` flag > persisted endpoint > error with guidance

**Why:** The operator does not create Routes (only the Console Proxy for browser access). The OLSConfig CR is cluster-scoped with a viewer role but contains no endpoint URL field. Adding auto-discovery would require operator changes (new status field or new RBAC) which is out of scope. The admin-creates-Route model is the established pattern.

**Alternatives considered:**
- Add endpoint to OLSConfig `.status.serviceURL` — rejected: requires operator changes, out of scope for CLI spec
- K8s API server proxy — rejected: requires service proxy RBAC grants
- Console proxy reuse — rejected: requires console URL discovery, adds complexity
- Environment variable only — rejected: per-context storage is better UX for multi-cluster users

### 4. Command Structure — Default Mode Dispatching

```
oc ols "question"              # default → ask mode
oc ols ask "question"          # explicit ask
oc ols troubleshoot "question" # explicit troubleshoot
oc ols config set-endpoint URL # configure endpoint
oc ols version                 # version info
```

**Why:** `oc ols "question"` is the simplest UX for the most common case. Cobra resolves subcommands first — if the first arg matches `ask`, `troubleshoot`, `config`, or `version`, it routes there. Otherwise the root command treats all args as the query and dispatches to `ask` mode. This avoids forcing `--mode=troubleshoot` on every invocation while keeping explicit mode selection available.

### 5. Conversation Persistence — Auto-Save with Transparency

Conversation IDs are auto-persisted per kubeconfig context. The CLI always informs the user when continuing a conversation:

```
$ oc ols "why is my pod crashing"
[answer...]

$ oc ols "what about the logs"
Continuing conversation a1b2c3d4...
[answer with context from previous question...]

$ oc ols --new "unrelated question"
[fresh conversation...]
```

**Storage:** `~/.config/oc-ols/contexts/<context-name>/conversation.json`

**Why:** The user explicitly requested that conversation continuation should be automatic but never hidden. The `--new` flag provides an escape hatch.

### 6. Terminal Markdown Rendering

LLM responses contain markdown. The CLI renders this using [glamour](https://github.com/charmbracelet/glamour) (charmbracelet) for terminal-readable output with ANSI styling.

- Default: rendered markdown (headings, code blocks, lists)
- `--output json`: bypasses rendering, returns raw structured response
- Non-TTY: auto-fallback to plain text (no ANSI codes when piped)

**Why:** Raw markdown in a terminal is hard to read. Glamour is the standard Go library for terminal markdown rendering, used by GitHub CLI (`gh`) and other tools.

### 7. Build & Distribution — Mirror Agentic CLI Pattern

Identical to `oc-agentic`:
- GoReleaser with 4 platform targets (linux/darwin × amd64/arm64)
- GitHub Actions workflow triggered on push to `main` (path-filtered)
- Rolling `latest` GitHub Release (overwritten each build)
- No semver, no brew, no RPM/DEB

**Why:** The pattern is proven, the infrastructure exists in the agentic operator, and consistency across CLI tools reduces maintenance burden.

## Command Reference

| Command | Description |
|---------|-------------|
| `oc ols "question"` | Ask in default (ask) mode with streaming |
| `oc ols ask "question"` | Explicit ask mode |
| `oc ols troubleshoot "question"` | Troubleshooting mode |
| `oc ols config set-endpoint URL` | Set service endpoint for current context |
| `oc ols version` | Print version |
| `--file path` | Attach file(s) — StringSlice, repeatable |
| `--new` | Start fresh conversation |
| `--conversation-id UUID` | Continue specific conversation |
| `--output json` | Full structured JSON response |
| `--endpoint URL` | Override endpoint for this invocation |
| `--insecure-skip-tls-verify` | Skip TLS verification |
| `--ca-cert path` | Custom CA certificate |

## Data Flow

```
User: oc ols "why is my pod crashing" --file pod.yaml
  │
  ├─ Load kubeconfig → bearer token + TLS config
  ├─ Resolve endpoint (flag > persisted > error)
  ├─ Load persisted conversation_id
  ├─ Read file attachments
  │
  ├─ POST /v1/streaming_query
  │    Authorization: Bearer <token>
  │    Body: {query, mode: "ask", conversation_id, attachments, media_type: "application/json"}
  │
  ├─ SSE stream:
  │    token events → raw text to stdout
  │    end event → conversation_id, referenced_documents
  │
  ├─ Render complete response via glamour
  ├─ Display referenced documents
  └─ Persist conversation_id
```

## Risks

| Risk | Mitigation |
|------|------------|
| Endpoint configuration UX friction | Clear first-run error message with exact command to run |
| Token expiry during long responses | Standard k8s token refresh; error message guides user to `oc login` |
| Glamour rendering issues with complex markdown | `--output json` bypass; non-TTY auto-fallback |
| SSE stream interruption | Partial output + warning to stderr |

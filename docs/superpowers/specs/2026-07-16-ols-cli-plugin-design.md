# OLS Chat CLI Plugin (oc-ols)

Design for a kubectl/oc CLI plugin that provides command-line access to OpenShift Lightspeed chat, enabling admins and users to query OLS without the web console.

**Jira**: OLS-1062
**Spec files**: [lightspeed-operator#1840](https://github.com/openshift/lightspeed-operator/pull/1840) (`cli.md`, `cli-distribution.md`)

## Scope

**In scope:**

- One-shot streaming chat via `oc ols "question"` (default ask mode)
- Explicit mode selection: `oc ols ask`, `oc ols troubleshoot`
- File attachments via `--file` (logs, YAML configs for troubleshooting)
- Auto-persisted conversation context per kubeconfig context with transparency
- Endpoint configuration per kubeconfig context (`oc ols config set-endpoint`)
- Terminal markdown rendering (glamour) for formatted LLM output
- Structured JSON output (`--output json`) for scripting
- Cross-platform builds (linux/darwin × amd64/arm64) via GoReleaser
- GitHub Actions CI with rolling `latest` release

**Not in scope:**

- Conversation history management (list/get/delete) — backend is cache-only
- Operator changes for service discovery or route auto-creation
- Interactive/REPL chat mode
- Windows binaries, Homebrew, RPM/DEB packaging

## Design Decisions

### 1. Repository & Language — Go in lightspeed-operator

The CLI lives in `lightspeed-operator` as a Go binary using cobra and client-go.

- Mirrors the pattern where `oc-agentic` lives in `lightspeed-agentic-operator`
- The operator repo already has Go toolchain and manages OLS deployment
- Go enables kubectl plugin discoverability (`oc-` prefix) and cross-platform binary distribution
- Python was rejected for inconsistency with `oc-agentic` and harder binary distribution

### 2. Architecture — REST API client, not K8s CRD client

Unlike `oc-agentic` (which uses controller-runtime for CRD CRUD), `oc-ols` is a pure HTTP client:

- **client-go** for kubeconfig reading only: bearer token extraction, TLS config, context resolution
- **net/http** for REST API calls to lightspeed-service `/v1/streaming_query`
- **No controller-runtime**, no CRD types, no scheme registration, no API server calls

This was validated via a contrarian challenge: "What if you didn't need a compiled Go binary at all?" The REST-only nature was acknowledged, but Go is still justified for kubeconfig integration, kubectl plugin discoverability, and distribution consistency.

| Aspect | oc-agentic | oc-ols |
|--------|-----------|--------|
| K8s client | controller-runtime typed + dynamic + clientset | client-go only (kubeconfig/token/TLS) |
| API target | K8s API server (AgenticRun CRDs) | lightspeed-service REST API |
| Command depth | `run {create,list,get,...}` + system commands | `{ask,troubleshoot}` + config + version |
| State | Stateless (reads CRDs) | Conversation persistence (local files) |
| Service discovery | N/A (talks to K8s API) | Admin-created Route, user-configured endpoint |

### 3. Service Discovery — Admin-created Route + user-configured endpoint

This is the most architecturally significant decision. The operator does NOT auto-discover or expose the service endpoint:

- The operator creates a `Service` (lightspeed-app-server:8443) but no `Route`
- The console accesses the service via a `ConsolePlugin` proxy (browser-only, not CLI-compatible)
- The `OLSConfig` CR (cluster-scoped) has no endpoint URL field

**Resolution:** The admin creates an OpenShift Route manually (per existing documentation). The user configures the CLI:

```
oc ols config set-endpoint https://lightspeed.apps.cluster.example.com
```

The endpoint is stored per kubeconfig context in `~/.config/oc-ols/contexts/<context-name>/endpoint`.

**Resolution order:** `--endpoint` flag > persisted endpoint > error with guidance.

**Alternatives rejected:**
- Add endpoint to OLSConfig status — requires operator changes, out of scope
- K8s API server proxy — requires service proxy RBAC grants
- Console proxy reuse — browser-only mechanism
- Environment variable only — poor UX across multiple clusters

### 4. Command Structure — Default mode dispatching

```
oc ols "question"                          # default ask mode
oc ols ask "question"                      # explicit ask mode
oc ols troubleshoot "question"             # troubleshoot mode
oc ols config set-endpoint <URL>           # per-context endpoint
oc ols version                             # version info

Global flags:
  --endpoint <URL>          --file <path>           --conversation-id <UUID>
  --new                     --output json           --insecure-skip-tls-verify
  --ca-cert <path>          --kubeconfig <path>
```

Cobra resolves subcommands first. If the first positional arg doesn't match a registered subcommand (`ask`, `troubleshoot`, `config`, `version`), the root command treats all args as the query and dispatches to `ask` mode. This gives the simplest UX for the common case while supporting explicit mode selection.

### 5. Conversation Persistence — Auto-persist with transparency

- After each query, the returned `conversation_id` is saved per kubeconfig context
- Subsequent queries auto-continue the conversation
- CLI always prints `"Continuing conversation <id>..."` to stderr — never hidden
- `--new` starts a fresh conversation; `--conversation-id <UUID>` overrides
- Storage: `~/.config/oc-ols/contexts/<context-name>/conversation.json`
- No automatic cleanup; users delete the file to reset

### 6. Output Rendering — Terminal markdown via glamour

LLM responses contain markdown (headings, code blocks, lists). The CLI renders this using [glamour](https://github.com/charmbracelet/glamour) (charmbracelet):

- Default: rendered markdown with ANSI styling, referenced documents displayed after the response
- `--output json`: bypasses rendering, returns full structured LLMResponse
- Non-TTY: glamour auto-falls back to plain text when piped

**Streaming consideration:** Tokens arrive incrementally via SSE. Raw tokens are streamed to stdout. The complete response is rendered through glamour after streaming completes.

### 7. SSE Event Handling

The CLI consumes lightspeed-service's `/v1/streaming_query` SSE stream:

| SSE Event | CLI Behavior |
|-----------|-------------|
| `start` | Internal: note stream begun |
| `token` | Print to stdout immediately |
| `reasoning` | Suppressed in default mode; captured for `--output json` |
| `tool_call` | Suppressed in default mode; captured for `--output json` |
| `end` | Persist conversation_id, display referenced documents |

### 8. Authentication & TLS

- Bearer token extracted from kubeconfig automatically (same as `oc whoami -t`)
- TLS settings inherited from kubeconfig context (CA cert, skip-verify)
- Override flags: `--insecure-skip-tls-verify`, `--ca-cert <path>`
- Auth failures (401/403) produce clear error messages with remediation guidance

### 9. Build & Distribution — Mirror agentic pattern

Identical to `oc-agentic`:

- GoReleaser with 4 targets (linux/darwin × amd64/arm64)
- GitHub Actions on push to main, path-filtered for CLI changes
- Rolling `latest` GitHub Release, overwritten each build
- 4 tarballs (`oc-ols_{os}_{arch}.tar.gz`) + `checksums.txt`
- Version from `git describe --tags --always`, injected via ldflags
- No semver, no brew, no RPM/DEB

## Data Flow

```
User: oc ols "why is my pod crashing" --file pod.yaml
  │
  ├─ Cobra → ask mode Run()
  │    ├─ Load kubeconfig → bearer token + TLS config
  │    ├─ Resolve endpoint (flag > persisted > error)
  │    ├─ Load persisted conversation_id
  │    ├─ Read file attachments
  │    ├─ Print "Continuing conversation <id>..." (if continuing)
  │    ├─ POST /v1/streaming_query (Authorization: Bearer <token>)
  │    ├─ Stream SSE tokens → stdout
  │    ├─ Render complete response via glamour
  │    ├─ Display referenced documents
  │    └─ Persist conversation_id
  │
  └─ Output: rendered markdown + references (default) | LLMResponse JSON (--output json)
```

## Error Handling

| Error Class | User Message |
|-------------|-------------|
| No endpoint | `Error: No endpoint configured. Run: oc ols config set-endpoint <URL>` |
| Auth failure (401) | `Error: Authentication failed. Try: oc login` |
| Forbidden (403) | `Error: Access denied. Contact your cluster administrator.` |
| Network/TLS | `Error: Could not connect to <endpoint>: <detail>` |
| Prompt too long (413) | `Error: Query exceeds maximum length.` |
| SSE interrupted | Partial output + `Warning: Response may be incomplete` |

## Implementation Notes

- File names in the module map are planned — update spec during implementation
- Conversation persistence storage path (`~/.config/oc-ols/`) follows XDG convention; revisable during implementation
- The `glamour` rendering decision may need adjustment based on streaming UX testing — buffer-then-render vs. stream-raw tradeoff
- Cross-repo spec references to `lightspeed-service what/api.md` and `what/auth.md` should be verified during implementation

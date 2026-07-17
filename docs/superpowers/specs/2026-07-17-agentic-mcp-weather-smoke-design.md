# Agentic MCP Weather Smoke Demo

Manual e2e proof that agentic OLS sandbox MCP support works, plus a CLI-first walkthrough suitable for recording. Uses a public non-OpenShift weather MCP so results are outside cluster context.

**Jira**: OLS-0000 (no ticket)

## Goals

- Prove the sandbox can connect to an external MCP server configured via `AgenticRun.spec.tools.mcpServers`.
- Show an agent invoking an MCP tool and using the result in the run summary.
- Keep the path simple enough to record: `kubectl`/`oc` primary, console as optional spectator.

## Non-goals

- Shipping a canary MCP image.
- Updating operator examples / opening a product PR for manifests (optional follow-up).
- ToolHive or MCP Lifecycle Operator.
- Classic OLS (`lightspeed-service`) MCP path.

## Architecture

```
kubectl apply weather Deployment/Service
        │
        ▼
mcp-weather.openshift-lightspeed.svc:8080/mcp
        │  (in-cluster Streamable HTTP)
        ▼
AgenticRun.spec.tools.mcpServers
        │
        ▼
agentic-operator patches sandbox with LIGHTSPEED_MCP_SERVERS
        │
        ▼
sandbox agent calls mcp__weather__* tools
        │
        ▼
Open-Meteo (egress) → tool result → AgenticRun summary
```

Wiring already exists in-tree:

- Operator: `AgenticRun.spec.tools.mcpServers` → env `LIGHTSPEED_MCP_SERVERS` on the sandbox pod/template.
- Sandbox: parses that env and registers MCP servers with Claude / Gemini / OpenAI providers (Streamable HTTP).

## Components

### 1. Weather MCP (in-cluster)

| Field | Value |
|---|---|
| Namespace | `openshift-lightspeed` |
| Image | `dog830228/mcp_weather_server:0.6.1` |
| Args | `--mode streamable-http --host 0.0.0.0 --port 8080` |
| Service | `mcp-weather`, port `8080` |
| MCP URL | `http://mcp-weather.openshift-lightspeed.svc:8080/mcp` |
| Auth | none for this demo |

Rationale: public image, Streamable HTTP (required by the sandbox), no API key (Open-Meteo), non-OCP domain so answers are not confused with `oc`/`kubectl`.

### 2. AgenticRun

- Analysis-only run (or full workflow with auto-approval if policy allows).
- Agent: existing cluster agent (e.g. `smart`).
- `spec.tools.mcpServers`:

```yaml
mcpServers:
  - name: weather
    url: http://mcp-weather.openshift-lightspeed.svc:8080/mcp
    timeoutSeconds: 60
```

- Request text must instruct the agent to use **only** MCP weather tools (`mcp__weather__*`), not invent weather and not use unrelated tools.

Example request intent: current weather for Prague; report temperature, conditions, and which MCP tool was used.

### 3. Verification (demo checklist)

1. Weather Deployment is Ready.
2. Sandbox pod/template env contains `LIGHTSPEED_MCP_SERVERS` with the weather URL.
3. Sandbox logs show an MCP weather tool invocation.
4. AgenticRun summary contains concrete weather for the chosen city.
5. Values roughly match a direct Open-Meteo check for the same city (proof against pure hallucination).

Console AI Hub may be open as a spectator; CLI is the source of truth for the recording.

## Prerequisites

- Agentic OLS installed (operator, Agent, LLMProvider, ApprovalPolicy as needed).
- Cluster can pull `dog830228/mcp_weather_server:0.6.1` from Docker Hub.
- Egress from weather pod (and sandbox if it proxies differently) to Open-Meteo (`api.open-meteo.com`).
- Ability to create `AgenticRun`s and approve steps if policy requires approval.

## Failure modes

| Symptom | Likely cause |
|---|---|
| Weather pod `ImagePullBackOff` | registry/network / missing pull secret |
| Sandbox cannot reach MCP | wrong URL, NetworkPolicy, Service DNS |
| Agent invents weather / no tool call | weak prompt, MCP missing from `LIGHTSPEED_MCP_SERVERS`, provider MCP wiring broken |
| Tool call fails / timeout | egress blocked, wrong `/mcp` path, transport not streamable-http |

## Demo script (CLI)

1. Apply weather Deployment + Service.
2. Wait until the weather pod is Ready; optionally `curl` the in-cluster `/mcp` initialize from a debug pod.
3. Apply the `AgenticRun` with `tools.mcpServers` and the weather-only request.
4. Approve analysis if required.
5. Watch run status; inspect sandbox env and logs for MCP tool calls.
6. Show the summary; cross-check with Open-Meteo for the same city.
7. (Optional) show the same run in the console.

## Follow-ups (out of scope)

- Promote manifests into `lightspeed-agentic-operator/examples/setup/` (replace outdated `08-mcp-demo.yaml` notes).
- Stronger canary MCP (nonce tool) if correlative weather proof is insufficient.
)

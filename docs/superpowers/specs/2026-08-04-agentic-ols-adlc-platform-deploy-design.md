# OLS-3195: Agentic OLS ADLC Platform Deploy Bundle

## Problem

The team needs a complete, reusable bundle to install Agentic OLS on the ADLC ROSA cluster so anyone can stand up the platform for the ADLC effort. Generic deploy scripts already exist in `lightspeed-agentic-operator/hack/quickstart/`; what is missing is ADLC-specific packaging: which cluster, which components, OpenAI agent config, secret handling, and a home for future adapters/jobs.

## Goals

- Anyone on the team can deploy Agentic OLS (operator + sandbox config + console + OpenAI agent) to the ADLC cluster by following one runbook.
- ADLC-specific knowledge lives in `lightspeed-team-harness`; generic deploy mechanics stay in the operator repo.
- Layout leaves room for future adapters and CronJobs (e.g. PR-review AgenticRuns) without reshuffling.

## Non-goals

- OTEL collector, alerts adapter, Postgres audit backend
- Sample / smoke-test AgenticRun YAML or automated smoke test
- Vault automation for the OpenAI API key (manual step only)
- Image build/push (use Konflux `:main` defaults)
- GitOps / CI-CD for the platform itself
- Implementing adapters or jobs in this story (directories reserved only)

## Target environment

| Item | Value |
|------|--------|
| Cluster console | https://console-openshift-console.apps.rosa.adlc-rosa-hcp.flhq.p3.openshiftapps.com/dashboards |
| Namespace | `openshift-lightspeed` |
| LLM | OpenAI via `LLMProvider` + `Agent` |
| Credentials | Manual: create Secret from CI vault OpenAI key |

## Architecture

```
lightspeed-team-harness/
  agentic-ols-adlc/
    README.md                 # ADLC index: cluster, namespace, subdir map
    platform/                 # OLS-3195 deliverable
      README.md               # deploy runbook
      install.sh              # thin wrapper over operator quickstart
      uninstall.sh
      manifests/
        openai-agent.yaml     # LLMProvider + Agent (no AgenticRun)
    adapters/                 # reserved (future)
    jobs/                     # reserved (future CronJobs → AgenticRuns)
```

**Ownership split**

| Concern | Owner |
|---------|--------|
| How components deploy (CRDs, Deployments, RBAC, webhook) | `lightspeed-agentic-operator/hack/quickstart/` |
| ADLC cluster identity, component selection, OpenAI overlay, ops runbook | `lightspeed-team-harness/agentic-ols-adlc/` |

## Install flow

`platform/install.sh`:

1. Prerequisites: `oc` logged in with cluster-admin; `AGENTIC_OPERATOR_REPO` pointing at an operator checkout (default: sibling `../lightspeed-agentic-operator` if present).
2. Ensure namespace `openshift-lightspeed`.
3. Call operator scripts selectively (do **not** call full `install.sh`, which also deploys OTEL + alerts adapter):
   - `deploy-configmap.sh` (sandbox PodSpec; OTEL optional — skipped when collector absent)
   - `deploy-operator.sh` (CRDs, operator, ApprovalPolicy, webhook)
   - `deploy-console.sh`
4. Secret gate: print CI-vault instructions; require `llm-creds-openai` in the namespace (fail with clear message if missing). No vault CLI integration.
5. `oc apply -f manifests/openai-agent.yaml`.

Image overrides: pass-through `--operator-image`, `--sandbox-image`, `--console-image` (defaults = Konflux `:main` from operator scripts).

`platform/uninstall.sh`: delete OpenAI CRs (and optionally the secret); undeploy console, operator, configmap via operator undeploy scripts.

## Manifests

`manifests/openai-agent.yaml` mirrors `hack/quickstart/examples/openai.yaml`:

- `LLMProvider/openai` → secret `llm-creds-openai` (`OPENAI_API_KEY`)
- `Agent/default` → model `gpt-5.4`, high reasoning effort
- No AgenticRun resources

## Runbook content

**Top-level README:** purpose, cluster console URL, namespace, subdir map, pointer to `platform/README.md`.

**platform/README.md:**

1. Prerequisites
2. Manual secret from CI vault (`oc create secret generic llm-creds-openai …`)
3. `./install.sh`
4. Verify: operator Deployment ready, console plugin registered, `LLMProvider/openai` + `Agent/default` present
5. Uninstall
6. Ops notes: image overrides, secrets must not be committed, vault path/name documented as known

Exact vault secret path/name may be filled when known; until then the runbook says to obtain the OpenAI key from the team CI vault (manual).

## Jira acceptance criteria (updated)

Replace / clarify OLS-3195 ACs as:

- Operator is running and watching `openshift-lightspeed`
- Base CRDs are installed (Agent, LLMProvider, ApprovalPolicy, AgenticRun, and related result/approval CRDs — not legacy Proposal/SandboxTemplate names)
- Console plugin is deployed and registered
- At least one LLMProvider and Agent CR are configured with working OpenAI credentials (secret created manually from CI vault)
- A runbook exists covering: deployment, base resource creation, and secret management
- Deployment details are documented (cluster console URL, namespace, registry defaults, access)

**Removed:** smoke-test Proposal submit/approve/process AC.

## Alternatives considered

| Approach | Decision |
|----------|----------|
| A. Harness runbook + thin install wrapper + OpenAI overlay | **Chosen** — reuses operator quickstart, ADLC-specific ops in harness |
| B. Kustomize overlay only, manual script calls | Rejected — worse one-command UX |
| C. Helm chart in harness | Rejected — duplicates quickstart; GitOps out of scope |

## Implementation notes

- Implementation lands in `lightspeed-team-harness` under `agentic-ols-adlc/`.
- Do not copy operator deploy scripts into harness; shell out to them.
- Update OLS-3195 description/ACs to match this design when implementing.
- `adapters/` and `jobs/` can be empty directories with a one-line README stub each.

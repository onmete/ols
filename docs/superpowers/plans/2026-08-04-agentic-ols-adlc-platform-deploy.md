# Agentic OLS ADLC Platform Deploy Bundle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an ADLC deploy bundle under `lightspeed-team-harness/agentic-ols-adlc/` that installs operator + sandbox config + console + OpenAI agent on the ADLC ROSA cluster via a thin wrapper over operator quickstart.

**Architecture:** Harness owns ADLC ops knowledge (cluster, component selection, OpenAI overlay, runbook). Operator repo owns deploy mechanics (`hack/quickstart/deploy-*.sh`). `platform/install.sh` shells out to those scripts, gates on a manually created OpenAI secret, then applies local manifests.

**Tech Stack:** Bash, `oc`, Kubernetes YAML (`agentic.openshift.io/v1alpha1`), Markdown runbooks

**Spec:** `docs/superpowers/specs/2026-08-04-agentic-ols-adlc-platform-deploy-design.md`

## Global Constraints

- Implementation lands only in `lightspeed-team-harness` (plus OLS-3195 Jira AC text update).
- Do not copy operator deploy scripts into harness; call them via `AGENTIC_OPERATOR_REPO`.
- Do not deploy OTEL, alerts adapter, or Postgres.
- Do not commit sample AgenticRun YAML or automate vault access.
- Namespace default: `openshift-lightspeed`.
- Cluster console: `https://console-openshift-console.apps.rosa.adlc-rosa-hcp.flhq.p3.openshiftapps.com/dashboards`
- Commit messages / PR titles: `OLS-3195 …` (fork-based PR against `openshift/lightspeed-team-harness`).

---

## File structure

| Path | Responsibility |
|------|----------------|
| `agentic-ols-adlc/README.md` | ADLC index: purpose, cluster, namespace, subdir map |
| `agentic-ols-adlc/platform/README.md` | Deploy/uninstall/verify runbook |
| `agentic-ols-adlc/platform/install.sh` | Thin install wrapper |
| `agentic-ols-adlc/platform/uninstall.sh` | Teardown wrapper |
| `agentic-ols-adlc/platform/manifests/openai-agent.yaml` | LLMProvider + Agent |
| `agentic-ols-adlc/adapters/README.md` | Stub for future adapters |
| `agentic-ols-adlc/jobs/README.md` | Stub for future CronJobs |
| `README.md` (harness root) | One-line pointer to `agentic-ols-adlc/` |

---

### Task 1: Scaffold directory tree and OpenAI manifest

**Files:**
- Create: `lightspeed-team-harness/agentic-ols-adlc/adapters/README.md`
- Create: `lightspeed-team-harness/agentic-ols-adlc/jobs/README.md`
- Create: `lightspeed-team-harness/agentic-ols-adlc/platform/manifests/openai-agent.yaml`

**Interfaces:**
- Consumes: none
- Produces: `openai-agent.yaml` with `LLMProvider/openai` + `Agent/default`; secret name `llm-creds-openai`

- [ ] **Step 1: Create reserved stubs**

```markdown
# adapters/

Future ADLC event adapters (Jira, GitHub, etc.). Not implemented yet.
```

```markdown
# jobs/

Future CronJobs / workloads that create AgenticRuns (e.g. PR review). Not implemented yet.
```

Write those to `agentic-ols-adlc/adapters/README.md` and `agentic-ols-adlc/jobs/README.md`.

- [ ] **Step 2: Create OpenAI manifest**

Write `agentic-ols-adlc/platform/manifests/openai-agent.yaml`:

```yaml
# OpenAI (direct API) for ADLC platform.
#
# Before applying, create the credentials secret:
#   oc create secret generic llm-creds-openai -n openshift-lightspeed \
#     --from-literal=OPENAI_API_KEY=sk-...
apiVersion: agentic.openshift.io/v1alpha1
kind: LLMProvider
metadata:
  name: openai
spec:
  type: OpenAI
  openAI:
    credentialsSecret:
      name: llm-creds-openai
---
apiVersion: agentic.openshift.io/v1alpha1
kind: Agent
metadata:
  name: default
spec:
  llmProvider:
    name: openai
  model: "gpt-5.4"
  reasoningConfig:
    effort: "high"
    summary: "auto"
  timeouts:
    analysisSeconds: 120
    executionSeconds: 120
    verificationSeconds: 120
```

- [ ] **Step 3: Verify manifest shape**

Run:

```bash
grep -E 'kind: (LLMProvider|Agent|AgenticRun)' \
  lightspeed-team-harness/agentic-ols-adlc/platform/manifests/openai-agent.yaml
```

Expected: lines for `LLMProvider` and `Agent` only — no `AgenticRun`.

- [ ] **Step 4: Commit**

```bash
cd lightspeed-team-harness
git add agentic-ols-adlc/adapters/README.md \
  agentic-ols-adlc/jobs/README.md \
  agentic-ols-adlc/platform/manifests/openai-agent.yaml
git commit -m "$(cat <<'EOF'
OLS-3195 scaffold agentic-ols-adlc tree and OpenAI manifest

EOF
)"
```

---

### Task 2: `install.sh` wrapper

**Files:**
- Create: `lightspeed-team-harness/agentic-ols-adlc/platform/install.sh`

**Interfaces:**
- Consumes: `$AGENTIC_OPERATOR_REPO/hack/quickstart/{deploy-configmap,deploy-operator,deploy-console}.sh`; `manifests/openai-agent.yaml`
- Produces: executable `install.sh` with flags `--operator-image`, `--sandbox-image`, `--console-image`, `--operator-repo`, `-h`

- [ ] **Step 1: Write `install.sh`**

Create `lightspeed-team-harness/agentic-ols-adlc/platform/install.sh` (mode `0755`) with this content:

```bash
#!/usr/bin/env bash
#
# ADLC platform installer for Agentic OLS.
# Deploys configmap + operator + console, then OpenAI LLMProvider/Agent.
# Does NOT deploy OTEL collector or alerts adapter.
#
# Usage:
#   ./install.sh
#   ./install.sh --operator-repo=/path/to/lightspeed-agentic-operator
#   AGENTIC_OPERATOR_REPO=... ./install.sh --sandbox-image=quay.io/...:tag
#
# Prerequisites:
#   - oc logged into the ADLC cluster with cluster-admin
#   - Secret llm-creds-openai in openshift-lightspeed (manual, from CI vault)
#   - A checkout of lightspeed-agentic-operator

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
HARNESS_ROOT="$(cd "${SCRIPT_DIR}/../.." && pwd)"
NAMESPACE="${NAMESPACE:-openshift-lightspeed}"
MANIFEST="${SCRIPT_DIR}/manifests/openai-agent.yaml"

OPERATOR_IMAGE=""
SANDBOX_IMAGE=""
CONSOLE_IMAGE=""
OPERATOR_REPO="${AGENTIC_OPERATOR_REPO:-}"

usage() {
  cat <<'EOF'
Usage: ./install.sh [options]

Options:
  --operator-repo=PATH   Path to lightspeed-agentic-operator checkout
                         (or set AGENTIC_OPERATOR_REPO)
  --operator-image=IMAGE Override agentic operator image
  --sandbox-image=IMAGE  Override sandbox image in ConfigMap PodSpec
  --console-image=IMAGE  Override console plugin image
  -h, --help             Show this help and exit
EOF
}

while [ $# -gt 0 ]; do
  case "$1" in
    --operator-repo=*)  OPERATOR_REPO="${1#*=}"; shift ;;
    --operator-repo)    [ $# -lt 2 ] && { echo "Missing value for $1" >&2; exit 1; }; OPERATOR_REPO="$2"; shift 2 ;;
    --operator-image=*) OPERATOR_IMAGE="${1#*=}"; shift ;;
    --operator-image)   [ $# -lt 2 ] && { echo "Missing value for $1" >&2; exit 1; }; OPERATOR_IMAGE="$2"; shift 2 ;;
    --sandbox-image=*)  SANDBOX_IMAGE="${1#*=}"; shift ;;
    --sandbox-image)    [ $# -lt 2 ] && { echo "Missing value for $1" >&2; exit 1; }; SANDBOX_IMAGE="$2"; shift 2 ;;
    --console-image=*)  CONSOLE_IMAGE="${1#*=}"; shift ;;
    --console-image)    [ $# -lt 2 ] && { echo "Missing value for $1" >&2; exit 1; }; CONSOLE_IMAGE="$2"; shift 2 ;;
    -h|--help)          usage; exit 0 ;;
    *)                  echo "Unknown flag: $1 (try --help)" >&2; exit 1 ;;
  esac
done

info()  { echo "  ✓ $*"; }
step()  { echo ""; echo "=== $* ==="; }
fail()  { echo "  ✗ $*" >&2; exit 1; }

# Resolve operator repo: explicit > env > sibling of harness root
if [ -z "${OPERATOR_REPO}" ]; then
  CANDIDATE="$(cd "${HARNESS_ROOT}/.." && pwd)/lightspeed-agentic-operator"
  if [ -d "${CANDIDATE}/hack/quickstart" ]; then
    OPERATOR_REPO="${CANDIDATE}"
  fi
fi

[ -n "${OPERATOR_REPO}" ] || fail "Set AGENTIC_OPERATOR_REPO or pass --operator-repo=PATH to a lightspeed-agentic-operator checkout."
OPERATOR_REPO="$(cd "${OPERATOR_REPO}" && pwd)"
QUICKSTART="${OPERATOR_REPO}/hack/quickstart"

step "1/6 Checking prerequisites"

command -v oc >/dev/null 2>&1 || fail "oc CLI not found."
oc whoami >/dev/null 2>&1 || fail "Not logged into a cluster. Run: oc login ..."
info "Logged in as $(oc whoami)"

if ! oc auth can-i create clusterrolebindings >/dev/null 2>&1; then
  fail "Current user lacks cluster-admin privileges."
fi
info "cluster-admin privileges confirmed"

for script in deploy-configmap.sh deploy-operator.sh deploy-console.sh; do
  [ -f "${QUICKSTART}/${script}" ] || fail "Missing ${QUICKSTART}/${script}"
done
info "Operator quickstart scripts found at ${QUICKSTART}"

[ -f "${MANIFEST}" ] || fail "Missing manifest: ${MANIFEST}"

step "2/6 Ensuring namespace ${NAMESPACE}"

if oc get namespace "${NAMESPACE}" >/dev/null 2>&1; then
  info "Namespace already exists"
else
  oc create namespace "${NAMESPACE}"
  info "Namespace created"
fi

step "3/6 Deploying sandbox configuration"

if [ -n "${SANDBOX_IMAGE}" ]; then
  bash "${QUICKSTART}/deploy-configmap.sh" --sandbox-image="${SANDBOX_IMAGE}"
else
  bash "${QUICKSTART}/deploy-configmap.sh"
fi

step "4/6 Deploying agentic operator"

if [ -n "${OPERATOR_IMAGE}" ]; then
  bash "${QUICKSTART}/deploy-operator.sh" --image="${OPERATOR_IMAGE}"
else
  bash "${QUICKSTART}/deploy-operator.sh"
fi

step "5/6 Deploying console plugin"

if [ -n "${CONSOLE_IMAGE}" ]; then
  bash "${QUICKSTART}/deploy-console.sh" --image="${CONSOLE_IMAGE}"
else
  bash "${QUICKSTART}/deploy-console.sh"
fi

step "6/6 Applying OpenAI agent manifests"

if ! oc get secret llm-creds-openai -n "${NAMESPACE}" >/dev/null 2>&1; then
  cat <<EOF >&2

  ✗ Secret llm-creds-openai not found in ${NAMESPACE}.

  Create it manually from the team CI vault OpenAI API key, then re-run this script:

    oc create secret generic llm-creds-openai -n ${NAMESPACE} \\
      --from-literal=OPENAI_API_KEY='<key-from-ci-vault>'

  Components (configmap, operator, console) are already deployed.
  Re-running ./install.sh after creating the secret is safe (idempotent).

EOF
  exit 1
fi
info "Secret llm-creds-openai present"

oc apply -f "${MANIFEST}"
info "LLMProvider/openai and Agent/default applied"

cat <<DONE

════════════════════════════════════════════════════════════════
  ADLC Agentic OLS platform installed.

  Namespace: ${NAMESPACE}
  Console:   https://console-openshift-console.apps.rosa.adlc-rosa-hcp.flhq.p3.openshiftapps.com/dashboards

  Verify:
    oc get deploy -n ${NAMESPACE}
    oc get llmprovider openai
    oc get agent default
    oc get consoleplugin

  Uninstall:
    ./uninstall.sh
════════════════════════════════════════════════════════════════
DONE
```

- [ ] **Step 2: Make executable and syntax-check**

```bash
chmod +x lightspeed-team-harness/agentic-ols-adlc/platform/install.sh
bash -n lightspeed-team-harness/agentic-ols-adlc/platform/install.sh
lightspeed-team-harness/agentic-ols-adlc/platform/install.sh --help
```

Expected: `bash -n` silent exit 0; `--help` prints Usage and exits 0.

- [ ] **Step 3: Commit**

```bash
cd lightspeed-team-harness
git add agentic-ols-adlc/platform/install.sh
git commit -m "$(cat <<'EOF'
OLS-3195 add ADLC platform install.sh wrapper

EOF
)"
```

---

### Task 3: `uninstall.sh` wrapper

**Files:**
- Create: `lightspeed-team-harness/agentic-ols-adlc/platform/uninstall.sh`

**Interfaces:**
- Consumes: `$AGENTIC_OPERATOR_REPO/hack/quickstart/{undeploy-console,undeploy-operator,undeploy-configmap}.sh`
- Produces: executable `uninstall.sh` with `--force`, `--keep-secret`, `--operator-repo`

- [ ] **Step 1: Write `uninstall.sh`**

Create `lightspeed-team-harness/agentic-ols-adlc/platform/uninstall.sh` (mode `0755`):

```bash
#!/usr/bin/env bash
#
# Tear down the ADLC Agentic OLS platform (console, operator, configmap, OpenAI CRs).
# Does not remove OTEL/alerts (ADLC install never deploys them).
#
# Usage:
#   ./uninstall.sh
#   ./uninstall.sh --force
#   ./uninstall.sh --keep-secret

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
HARNESS_ROOT="$(cd "${SCRIPT_DIR}/../.." && pwd)"
NAMESPACE="${NAMESPACE:-openshift-lightspeed}"
FORCE=0
KEEP_SECRET=0
OPERATOR_REPO="${AGENTIC_OPERATOR_REPO:-}"

usage() {
  cat <<'EOF'
Usage: ./uninstall.sh [options]

Options:
  --operator-repo=PATH   Path to lightspeed-agentic-operator checkout
  --keep-secret          Do not delete llm-creds-openai
  --force                Skip confirmation prompt
  -h, --help             Show this help and exit
EOF
}

while [ $# -gt 0 ]; do
  case "$1" in
    --operator-repo=*) OPERATOR_REPO="${1#*=}"; shift ;;
    --operator-repo)   [ $# -lt 2 ] && { echo "Missing value for $1" >&2; exit 1; }; OPERATOR_REPO="$2"; shift 2 ;;
    --keep-secret)     KEEP_SECRET=1; shift ;;
    --force)           FORCE=1; shift ;;
    -h|--help)         usage; exit 0 ;;
    *)                 echo "Unknown flag: $1 (try --help)" >&2; exit 1 ;;
  esac
done

info()  { echo "  ✓ $*"; }
step()  { echo ""; echo "=== $* ==="; }
fail()  { echo "  ✗ $*" >&2; exit 1; }

if [ -z "${OPERATOR_REPO}" ]; then
  CANDIDATE="$(cd "${HARNESS_ROOT}/.." && pwd)/lightspeed-agentic-operator"
  if [ -d "${CANDIDATE}/hack/quickstart" ]; then
    OPERATOR_REPO="${CANDIDATE}"
  fi
fi

[ -n "${OPERATOR_REPO}" ] || fail "Set AGENTIC_OPERATOR_REPO or pass --operator-repo=PATH."
OPERATOR_REPO="$(cd "${OPERATOR_REPO}" && pwd)"
QUICKSTART="${OPERATOR_REPO}/hack/quickstart"

for script in undeploy-console.sh undeploy-operator.sh undeploy-configmap.sh; do
  [ -f "${QUICKSTART}/${script}" ] || fail "Missing ${QUICKSTART}/${script}"
done

if [ "${FORCE}" != "1" ]; then
  echo "This will delete ADLC OpenAI CRs, console plugin, operator, and configmap"
  echo "in namespace ${NAMESPACE}."
  if [ "${KEEP_SECRET}" != "1" ]; then
    echo "Secret llm-creds-openai will also be deleted (pass --keep-secret to retain)."
  fi
  echo ""
  read -rp "Continue? [y/N] " confirm
  case "${confirm}" in
    [yY][eE][sS]|[yY]) ;;
    *) echo "Aborted."; exit 0 ;;
  esac
fi

step "1/5 Deleting OpenAI Agent and LLMProvider"

oc delete agent default --ignore-not-found >/dev/null 2>&1 || true
oc delete llmprovider openai --ignore-not-found >/dev/null 2>&1 || true
info "OpenAI CRs deleted"

step "2/5 Credential secret"

if [ "${KEEP_SECRET}" = "1" ]; then
  info "Keeping llm-creds-openai (--keep-secret)"
else
  oc delete secret llm-creds-openai -n "${NAMESPACE}" --ignore-not-found >/dev/null 2>&1 || true
  info "Secret llm-creds-openai deleted (if present)"
fi

step "3/5 Removing console plugin"
bash "${QUICKSTART}/undeploy-console.sh"

step "4/5 Removing agentic operator"
bash "${QUICKSTART}/undeploy-operator.sh"

step "5/5 Removing configuration"
bash "${QUICKSTART}/undeploy-configmap.sh"

echo ""
info "ADLC platform uninstall complete."
```

- [ ] **Step 2: Make executable and syntax-check**

```bash
chmod +x lightspeed-team-harness/agentic-ols-adlc/platform/uninstall.sh
bash -n lightspeed-team-harness/agentic-ols-adlc/platform/uninstall.sh
lightspeed-team-harness/agentic-ols-adlc/platform/uninstall.sh --help
```

Expected: exit 0; Usage printed.

- [ ] **Step 3: Commit**

```bash
cd lightspeed-team-harness
git add agentic-ols-adlc/platform/uninstall.sh
git commit -m "$(cat <<'EOF'
OLS-3195 add ADLC platform uninstall.sh wrapper

EOF
)"
```

---

### Task 4: Runbooks and harness README pointer

**Files:**
- Create: `lightspeed-team-harness/agentic-ols-adlc/README.md`
- Create: `lightspeed-team-harness/agentic-ols-adlc/platform/README.md`
- Modify: `lightspeed-team-harness/README.md`

**Interfaces:**
- Consumes: install/uninstall scripts and cluster URL from spec
- Produces: human-readable deploy path starting from harness root README

- [ ] **Step 1: Write top-level ADLC README**

Create `lightspeed-team-harness/agentic-ols-adlc/README.md`:

```markdown
# Agentic OLS — ADLC

Team ADLC platform: Agentic OLS on the shared ROSA cluster.

## Environment

| Item | Value |
|------|--------|
| Console | https://console-openshift-console.apps.rosa.adlc-rosa-hcp.flhq.p3.openshiftapps.com/dashboards |
| Namespace | `openshift-lightspeed` |
| LLM | OpenAI (`LLMProvider/openai` + `Agent/default`) |

Log in with `oc` (ROSA / SSO) before deploying. You need cluster-admin (or equivalent) for the first install.

## Layout

| Path | Purpose |
|------|---------|
| [`platform/`](platform/) | Deploy/uninstall Agentic OLS (operator, sandbox config, console, OpenAI agent) |
| [`adapters/`](adapters/) | Reserved — future event adapters |
| [`jobs/`](jobs/) | Reserved — future CronJobs that create AgenticRuns |

## Start here

See [`platform/README.md`](platform/README.md) for the deploy runbook.
```

- [ ] **Step 2: Write platform runbook**

Create `lightspeed-team-harness/agentic-ols-adlc/platform/README.md`:

```markdown
# ADLC platform deploy runbook

Installs Agentic OLS for the ADLC effort: sandbox ConfigMap, operator (CRDs + controllers), console plugin, and OpenAI `LLMProvider`/`Agent`.

Does **not** deploy OTEL collector or alerts adapter.

## Prerequisites

- `oc` on PATH, logged into the ADLC ROSA cluster
- cluster-admin (needed for CRDs / cluster-scoped RBAC)
- A local checkout of [`lightspeed-agentic-operator`](https://github.com/openshift/lightspeed-agentic-operator)
  - Default resolution: sibling of this harness repo at `../lightspeed-agentic-operator`
  - Or set `AGENTIC_OPERATOR_REPO` / pass `--operator-repo=`
- OpenAI API key from the team CI vault (manual; see Secret step)

## 1. Create the OpenAI secret

Obtain the OpenAI API key from the team CI vault (ask a teammate if you do not have vault access). Do **not** commit the key.

```bash
oc create namespace openshift-lightspeed --dry-run=client -o yaml | oc apply -f -

oc create secret generic llm-creds-openai -n openshift-lightspeed \
  --from-literal=OPENAI_API_KEY='<key-from-ci-vault>'
```

If the secret already exists, skip this step (or delete/recreate if rotating).

## 2. Install

From this directory:

```bash
./install.sh
```

Optional image overrides:

```bash
./install.sh \
  --operator-image=quay.io/redhat-user-workloads/crt-nshift-lightspeed-tenant/lightspeed-agentic-operator:main \
  --sandbox-image=quay.io/redhat-user-workloads/crt-nshift-lightspeed-tenant/lightspeed-agentic-sandbox:main \
  --console-image=quay.io/redhat-user-workloads/crt-nshift-lightspeed-tenant/lightspeed-agentic-console:main
```

If the secret is missing, install still deploys configmap/operator/console, then exits with instructions. Create the secret and re-run `./install.sh`.

## 3. Verify

```bash
oc get deploy -n openshift-lightspeed
oc get llmprovider openai
oc get agent default
oc get consoleplugin
```

Expect the agentic operator Deployment Available, `LLMProvider/openai` and `Agent/default` present, and a console plugin for agentic OLS registered.

## 4. Uninstall

```bash
./uninstall.sh
# or non-interactive:
./uninstall.sh --force

# keep the OpenAI secret:
./uninstall.sh --force --keep-secret
```

## Ops notes

- Secrets must never be committed to git.
- Images default to Konflux `:main` tags from the operator quickstart scripts.
- Generic deploy mechanics live in the operator repo under `hack/quickstart/`; this directory only selects ADLC components and applies the OpenAI overlay.
- Cluster console: https://console-openshift-console.apps.rosa.adlc-rosa-hcp.flhq.p3.openshiftapps.com/dashboards
```

- [ ] **Step 3: Point harness root README at ADLC**

After the opening paragraph in `lightspeed-team-harness/README.md` (after “Shared AI coding skills…”), add:

```markdown
## Agentic OLS ADLC

Deploy and operate the team ADLC Agentic OLS platform: [`agentic-ols-adlc/`](agentic-ols-adlc/).
```

- [ ] **Step 4: Commit**

```bash
cd lightspeed-team-harness
git add agentic-ols-adlc/README.md \
  agentic-ols-adlc/platform/README.md \
  README.md
git commit -m "$(cat <<'EOF'
OLS-3195 document ADLC deploy runbook and harness entrypoint

EOF
)"
```

---

### Task 5: Update OLS-3195 acceptance criteria in Jira

**Files:**
- Jira: `OLS-3195` (no repo files)

**Interfaces:**
- Consumes: updated ACs from the design spec
- Produces: Jira description matching what the harness bundle delivers

- [ ] **Step 1: Edit OLS-3195 description**

Replace the Acceptance Criteria section with:

```markdown
## Acceptance Criteria

* Operator is running and watching `openshift-lightspeed`
* Base CRDs are installed (Agent, LLMProvider, ApprovalPolicy, AgenticRun, and related result/approval CRDs)
* Console plugin is deployed and registered
* At least one LLMProvider and Agent CR are configured with working OpenAI credentials (secret created manually from CI vault)
* A runbook exists in `lightspeed-team-harness/agentic-ols-adlc/platform/` covering deployment, base resource creation, and secret management
* Deployment details are documented (ROSA ADLC HCP console URL, namespace `openshift-lightspeed`, Konflux image defaults, access)

## Notes

* Smoke-test AgenticRun is out of scope for this story
* OTEL collector and alerts adapter are not part of the ADLC platform install
* Bundle location: `lightspeed-team-harness/agentic-ols-adlc/`
```

Use Jira MCP `editJiraIssue` (or UI). Keep the user story paragraph; refresh ACs/notes as above.

- [ ] **Step 2: Confirm**

Open https://redhat.atlassian.net/browse/OLS-3195 and verify ACs match.

---

### Task 6: Open harness PR

**Files:**
- All files from Tasks 1–4 in `lightspeed-team-harness`

- [ ] **Step 1: Squash / ensure clean history on a feature branch**

```bash
cd lightspeed-team-harness
git checkout -b OLS-3195-agentic-ols-adlc-platform
# if commits already made on another branch, cherry-pick or reset --soft as needed
git log --oneline origin/main..HEAD
```

Prefer one squash commit before push (per harness AGENTS.md):

```bash
git reset --soft $(git merge-base HEAD origin/main)
git commit -m "$(cat <<'EOF'
OLS-3195 add agentic-ols-adlc platform deploy bundle

Thin install/uninstall wrappers over operator quickstart, OpenAI
agent manifests, and ADLC runbook for the ROSA cluster.
EOF
)"
```

- [ ] **Step 2: Push fork and open PR**

```bash
# detect fork remote (username from gh api user)
git push <fork-remote> OLS-3195-agentic-ols-adlc-platform
gh pr create --repo openshift/lightspeed-team-harness \
  --head <fork-user>:OLS-3195-agentic-ols-adlc-platform \
  --base main \
  --title "OLS-3195 add agentic-ols-adlc platform deploy bundle" \
  --body "$(cat <<'EOF'
## Summary
- Add `agentic-ols-adlc/platform` install/uninstall wrappers over operator quickstart (configmap + operator + console; no OTEL/alerts)
- OpenAI LLMProvider/Agent manifests and ADLC runbook for ROSA cluster
- Reserve `adapters/` and `jobs/` for follow-up ADLC work

## Test plan
- [ ] `bash -n` on install.sh and uninstall.sh
- [ ] `./install.sh --help` / `./uninstall.sh --help`
- [ ] On ADLC cluster (with secret): `./install.sh` then verify deploy/llmprovider/agent/consoleplugin
- [ ] `./uninstall.sh --force` tears down cleanly

EOF
)"
```

- [ ] **Step 3: Return PR URL**

Record the PR URL in the OLS-3195 Jira comment.

---

## Spec coverage checklist

| Spec requirement | Task |
|------------------|------|
| `agentic-ols-adlc/` layout with platform/adapters/jobs | 1, 4 |
| Top-level + platform runbooks, cluster URL, namespace | 4 |
| `install.sh` selective deploy + secret gate + OpenAI apply | 2 |
| `uninstall.sh` | 3 |
| `openai-agent.yaml` without AgenticRun | 1 |
| No OTEL/alerts | 2 (scripts called) |
| Manual CI vault secret | 2, 4 |
| Update OLS-3195 ACs | 5 |
| Harness PR | 6 |

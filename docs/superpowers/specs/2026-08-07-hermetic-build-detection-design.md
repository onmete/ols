# OLS-3413: Replace cachi2.env file check with build arg in Containerfiles

## Problem

All OLS Python Containerfiles detect hermetic Konflux builds by checking for the file `/cachi2/cachi2.env`. This is an undocumented implementation detail of the Konflux build CLI — the file was removed in buildah task 0.10, breaking all hermetic builds. Konflux re-added it temporarily (build-definitions#3649) but confirmed the file is not a public API and may be removed again.

## Solution

Replace the file-based detection with an explicit `HERMETIC_BUILD` build arg set by the Tekton pipeline. Remove all sourcing of `cachi2.env` since Konflux's build CLI already injects `PIP_FIND_LINKS` and `PIP_NO_INDEX` as environment variables on every `RUN` instruction via buildah secret env mounts (buildah >= 1.44.0) or Containerfile rewriting (older buildah).

## Design

### Detection mechanism

Each Containerfile declares a build arg with a safe default:

```dockerfile
ARG HERMETIC_BUILD=false
```

All conditional checks change from:

```dockerfile
# Before
RUN if [ -f /cachi2/cachi2.env ]; then ...
RUN if [ ! -f /cachi2/cachi2.env ]; then ...

# After
RUN if [ "${HERMETIC_BUILD}" = "true" ]; then ...
RUN if [ "${HERMETIC_BUILD}" != "true" ]; then ...
```

### Pip env vars

The `. /cachi2/cachi2.env` sourcing line is removed. The hermetic branch uses `${PIP_FIND_LINKS}` directly — Konflux's build CLI sets it before any `RUN` instruction executes. No fallback sourcing is needed.

### Tekton pipeline changes

Each repo's `.tekton/` pipeline YAML adds `HERMETIC_BUILD=true` to the build args for pipelines that run hermetic builds. The mechanism depends on how the repo currently passes build args (inline `build-args` array or `build-args-file`).

## Affected repos

| Repo | File | Occurrences |
|------|------|-------------|
| lightspeed-service | `Containerfile` | 2 (lines 34, 63) |
| lightspeed-service | `.tekton/*.yaml` | Add build arg |
| lightspeed-agentic-sandbox | `Containerfile` | 1 (line 35) |
| lightspeed-agentic-sandbox | `.tekton/*.yaml` | Add build arg |

Other repos (lightspeed-operator, lightspeed-console, lightspeed-hub, lightspeed-agentic-operator) have no `cachi2` references in their Containerfiles.

## Changes per repo

### Containerfile

1. Add `ARG HERMETIC_BUILD=false` (in the builder stage, before first use)
2. Replace `if [ -f /cachi2/cachi2.env ]` with `if [ "${HERMETIC_BUILD}" = "true" ]`
3. Replace `if [ ! -f /cachi2/cachi2.env ]` with `if [ "${HERMETIC_BUILD}" != "true" ]`
4. Remove `. /cachi2/cachi2.env &&` sourcing line
5. Remove comments referencing `cachi2.env`

### Tekton pipelines

1. Add `HERMETIC_BUILD=true` to the build args for push and pull-request pipelines that set `hermetic: 'true'`

## Rollout

Each repo gets its own PR since they are separate repositories with independent CI. No ordering dependency between the PRs — each is self-contained.

## References

- Jira: [OLS-3413](https://redhat.atlassian.net/browse/OLS-3413)
- Konflux fix (temporary): [build-definitions#3649](https://github.com/konflux-ci/build-definitions/pull/3649)
- Konflux build CLI env injection: [konflux-build-cli build.go](https://github.com/konflux-ci/konflux-build-cli/blob/main/pkg/commands/build.go) — `setPrefetchEnvSecrets()` (buildah >= 1.44.0) and `injectPrefetchEnvToContainerfile()` (fallback)

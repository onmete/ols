# OLS-3450: Credential Hot-Reload

**Jira:** [OLS-3450](https://redhat.atlassian.net/browse/OLS-3450)  
**Related:** RFE-9380, [service PR #2955](https://github.com/openshift/lightspeed-service/pull/2955), [operator PR #1779](https://github.com/openshift/lightspeed-operator/pull/1779)  
**Date:** 2026-09-01  
**Status:** Implemented (awaiting review)

## Problem

When an LLM credential secret rotates (e.g., automated key rotation by a vault operator), the lightspeed-operator's watcher detects the `.data` change, sets the `ols.openshift.io/force-reload` annotation on the pod template, and triggers a rolling restart of the app-server Deployment. This causes a brief downtime window on every rotation event — problematic for high-rotation environments (short-lived tokens, frequent rotations).

## Solution

An opt-in `spec.ols.credentialHotReload` boolean field on the `OLSConfig` CRD (default `false`). When enabled, the operator skips watching LLM credential secrets and the service re-reads credentials from disk on every request.

### Operator Behavior

When `credentialHotReload` is **true**:

1. During `annotateExternalResources()`, LLM credential secrets (those with source prefix `llm-provider-*`) are **not annotated** with `ols.openshift.io/watcher: cluster`. Instead, `removeSecretAnnotationIfNeeded()` is called, which removes the annotation if it already exists (from a previous reconcile with the flag disabled).
2. Because the secrets are not annotated, the existing watcher predicate (`SecretWatcherFilter`) never matches → `SecretUpdateHandler` never fires for LLM secrets → no pod restart on credential rotation. The watcher code itself is unchanged.
3. An info-level log is emitted during reconciliation when the flag is enabled: "credentialHotReload is enabled — LLM credential secret rotations will not trigger app-server restarts."
4. The operator writes `credential_hot_reload: true` into the generated `olsconfig.yaml` in the `ols_config` section (via `buildOLSConfig()`).

When `credentialHotReload` is **false** (default):

- Existing behavior is preserved: LLM credential secrets are annotated, watched, and trigger rolling restarts on `.data` change.
- `credential_hot_reload` is written as `false` (or omitted) in `olsconfig.yaml`.

### Non-LLM Secrets Are Unaffected

The following secrets are always annotated and always trigger restarts regardless of the `credentialHotReload` flag:

- Custom TLS secret (`spec.ols.tlsConfig.keyCertSecretRef`)
- MCP header secrets (`spec.mcpServers[].headers[].valueFrom.secretRef`)
- System secrets (pull secret, service-ca certs, PostgreSQL certs, kube-root-ca)

### CR-Level Secret Reference Changes Always Restart

Changing the `credentialsSecretRef.name` in the CR (i.e., pointing to a different secret) modifies the Deployment's volume mounts. This triggers a standard reconciliation and rolling restart regardless of `credentialHotReload`.

### Service Behavior

When `credential_hot_reload` is **true** in `olsconfig.yaml`:

1. `ProviderConfig` stores the credential file path in a Pydantic `PrivateAttr` (`_credentials_path`) and the flag in `_credential_hot_reload`, both set during `__init__`. The `credential_hot_reload` flag is propagated from the root `Config` → `LLMProviders` → each `ProviderConfig`.
2. `get_credentials()` re-reads from the file path via `read_secret_from_path()` on every call. If the fresh read succeeds, it updates the cached `self.credentials` and returns the new value. All 7 provider implementations call `get_credentials()` instead of accessing `self.credentials` directly.
3. `get_aws_credentials()` follows the same pattern for Bedrock IAM credentials (access key, secret key, and optional role ARN), re-reading each file.
4. `read_secret_from_path()` (new helper in `checks.py`) handles both file and directory paths (appending `default_filename` for directories). Returns `None` on `OSError` with a warning log.
5. Kubelet atomically updates the `..data` symlink when a mounted secret changes, so the next request reads the new value without a race.
6. If the file read fails (returns `None`), the last successfully read credential value is retained in `self.credentials` (graceful degradation).

When `credential_hot_reload` is **false** (default):

- Existing behavior: credentials are read once at startup and cached in `ProviderConfig.credentials`.

### Data Flow

```
DEFAULT (flag off):
  Secret.data change → watcher annotation match → force-reload → rolling restart → re-read at startup

HOT-RELOAD (flag on):
  Secret.data change → kubelet updates mount → service get_credentials() on next request
                      (no watcher annotation → no operator restart)
```

## CRD Change

New field on `OLSSpec`:

| Field path | JSON key | Go type | Required | Default | Description |
|---|---|---|---|---|---|
| `spec.ols.credentialHotReload` | `credentialHotReload` | `bool` | No | `false` | When true, the service re-reads LLM credentials from disk per request and the operator skips restarting the app-server on LLM secret rotation |

## Compatibility

- **Backward compatible**: default `false` preserves current behavior for all clusters.
- **Version skew**: enabling the flag with an older service image that lacks per-request re-read leaves rotated credentials stale until a manual restart. The operator emits a warning for this scenario.
- **No CRD breaking change**: additive field with default.

## Changes by Repository

| Repository | Files / Areas | Change |
|---|---|---|
| `lightspeed-operator` | `api/v1alpha1/olsconfig_types.go` (CRD type), `internal/controller/appserver/assets.go` (config gen), `internal/controller/olsconfig_helpers.go` (annotation skip + `removeSecretAnnotationIfNeeded()`), `internal/controller/olsconfig_controller.go` (info log), `internal/controller/utils/types.go` (YAML struct) | New `CredentialHotReload *bool` on `OLSSpec`, annotation skip for `llm-provider-*` sources, `credential_hot_reload` in `OLSConfig` YAML struct |
| `lightspeed-service` | `ols/app/models/config.py` (`ProviderConfig`, `LLMProviders`, `Config`), `ols/utils/checks.py`, 7 provider files | `get_credentials()` / `get_aws_credentials()` on `ProviderConfig`, `read_secret_from_path()` helper, all providers use `get_credentials()` |

## Acceptance Criteria

1. With `credentialHotReload: false` (default), rotating an LLM secret triggers an app-server rolling restart (existing behavior).
2. With `credentialHotReload: true`, rotating an LLM secret does **not** restart the app-server; the next request uses the new credential.
3. Rotating a non-LLM secret (TLS, MCP, system) always triggers a restart regardless of flag value.
4. Changing `credentialsSecretRef.name` in the CR triggers a restart regardless of flag value.
5. If the credential file read fails at request time, the service uses the last good value and logs a warning.
6. The operator emits a warning log during reconciliation when the flag is enabled.

## Testing Strategy

- **Operator unit tests**: verify annotation skip/removal logic, olsconfig.yaml generation with flag on/off, warning log emission.
- **Service unit tests**: verify `get_credentials()` re-reads file when flag on, uses cached value when flag off, falls back on read failure.
- **E2e test**: rotate a secret with flag enabled and verify the pod is not restarted and the service picks up the new credential.

## Risk Assessment

**Risk Level 3 (High)** — CRD field addition, changes credential handling semantics (security-adjacent), cross-repo coordination (operator + service must be compatible).

# Cross-Repo Constraints

Rules that apply across all repositories in the OLS workspace. Violating any of these means the system or process is wrong.

## Git Workflow

1. All repos use a fork-based workflow. Push to your fork, PR against `origin/main`.
2. All commit messages and PR titles start with `OLS-XXXX` (Jira ticket reference).
3. Squash commits before pushing — one logical commit per PR unless the PR explicitly tracks multiple independent changes.

## Jira

4. Project key is **OLS** on `redhat.atlassian.net`.

## Kubernetes

5. Classic OLS CRDs use API group `ols.openshift.io/v1alpha1`.
6. Agentic OLS CRDs use API group `agentic.openshift.io/v1alpha1`.
7. Multicluster Hub CRDs use API group `hub.openshift.io/v1alpha1`.
8. All components deploy into the `openshift-lightspeed` namespace.
9. Hub-managed spoke-side resources (per-step SAs, RBAC for multicluster) deploy into the `openshift-lightspeed-managed` namespace on the spoke. This avoids collisions with a spoke-local OLS installation in `openshift-lightspeed`.

## RAG

10. The embedding model used to build RAG indexes must be identical to the model used to query them at runtime. Model mismatch produces meaningless similarity scores.

## Version Support

11. Classic OLS is supported on all OCP versions. The agentic layer (agentic-operator, agentic console plugin, agentic sandbox, alerts adapter, and the classic operator's agentic operands/handoff) is installed only on OCP ≥ 5.0. Every OCP 4.x runs classic-only, with no agentic components, CRDs, or RBAC present. See decision `0037-agentic-version-gating.md`.

# bo-deploy

**The gate.** Rendered manifests only — CI writes, Argo CD reads.

Part of a three-cluster GitOps lab demonstrating progressive delivery gated on
**error-budget burn rate**. Spec: [`bo-platform/BUILD-PLAN.md`](https://github.com/bo-jr/bo-platform/blob/main/BUILD-PLAN.md) ·
Working rules: [`CLAUDE.md`](./CLAUDE.md)

Merging a PR here deploys to production. `rendered/{dev,prod}/<service>/manifests.yaml` is machine output; never hand-edit it — the required `validate` check refuses any change to `rendered/` that is not CI writing its own service. Branch protection on `main` is load-bearing.

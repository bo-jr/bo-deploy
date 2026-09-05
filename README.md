# bo-deploy

**The gate.** Rendered manifests only — CI writes, Argo CD reads.

Part of a three-cluster GitOps lab demonstrating progressive delivery gated on
**error-budget burn rate**. Spec: [`bo-platform/docs/BUILD-PLAN.md`](https://github.com/bo-jr/bo-platform/blob/main/docs/BUILD-PLAN.md) ·
Working rules: [`CLAUDE.md`](./CLAUDE.md)

Merging a PR here deploys to production. `rendered/{dev,prod}/<service>/` is machine output; never hand-edit it. Branch protection on `main` is load-bearing.

# bo-deploy — THE GATE

Rendered manifests only. **CI writes. Argo CD reads. Humans review and merge.**

Canonical spec: [`bo-platform/docs/BUILD-PLAN.md`](https://github.com/bo-jr/bo-platform/blob/main/docs/BUILD-PLAN.md)
§4 (why this repo is shared), Phase 3 (promotion), Phase 6 scenario 5 (atomicity).

## Read this before touching anything

This repo is the **only manual gate in the entire system**. Merging a PR here deploys to
production. Everything else in the lab — the canary analysis, the burn-rate SLO, the
soak, the reconciler — exists to make the decision at this merge button an informed one.

```
rendered/
├── dev/<service>/      <- written by GitHub Actions on merge to a service repo
└── prod/<service>/     <- written by cmd/promoter after the soak passes
```

## Never hand-edit `rendered/`

It is machine output. CI runs `helm template` and commits plain YAML, which is precisely
what makes the promotion PR diff **byte-identical to what gets applied**. Argo CD never
templates at sync time for services. Hand-editing breaks that guarantee silently: the
next CI run overwrites your change and nobody finds out why the fix disappeared.

To change what is rendered, change the source: the service's `chart-values.yaml`, or
`bo-service-chart`.

**Break-glass is not an exception to this.** To deploy a specific digest ahead of the
soak, open the prod PR **by hand with that digest** — doing manually exactly what
`cmd/promoter` does. There is deliberately no bypass flag and no emergency mode, because
a code path that only executes during incidents is a code path that is never tested.
The reconciler is idempotent and will leave your PR alone.

## Why one shared repo and not one per service

**Atomicity.** Phase 6 scenario 5 — the `catalog` schema migration — requires `catalog`
and `storefront` to promote **together or not at all**: one PR carrying both digests, one
merge, one decision. Across three config repos that is three coordinated merges with no
transaction, and the failure mode is prod running a half-promoted pair.

It also means branch protection is load-bearing in exactly one place. This one.

## Branch protection on `main` is not decoration

Required, and the reason all seven repos are **public** — on a private free repo these
rules are configured and then **silently not enforced**:

- require a pull request before merging
- require 1 approving review
- **require approval from someone other than the last pusher**
- dismiss stale approvals on new commits
- required status check: the dev-health check
- **a status check that re-queries dev health at merge time** — otherwise a PR opened
  Tuesday can be merged Thursday after dev has since degraded, and nothing catches it.
  This is the most commonly missed piece of a promotion pipeline.

Applied by `task repos:protect` from `bo-platform`, which loops one ruleset payload over
all seven via `gh api`. A personal account has no account-level rulesets, so this is a
seven-time setup. **`bo-deploy` is the one you would otherwise get wrong.**

## What promotion actually reads

`cmd/promoter` runs as a CronJob in `mgmt` every 5 minutes and asks what *should* be
true — it is **level-triggered, never edge-triggered**. No webhooks. A thousand runs
produce one PR; two hundred missed runs during a cluster pause cost nothing.

It reads the digest from the **live dev Rollout**, not from `rendered/dev/`. The manifest
is what *should* be running; the Rollout is what *is* running and what the analysis
actually verified. During an aborted rollout they diverge — and that is exactly the case
where you must promote what passed, not what was requested.

## Rolling PRs, force-pushed

One PR per service, force-pushed to the latest digest — not one PR per commit. Expect
`synchronize` events on already-open PRs; that noise is known and documented in Phase 4.

## What must never live here

- Source code, Dockerfiles, or Helm charts
- Hand-written YAML of any kind
- Kustomize overlays — CI renders; there is nothing left to overlay
- Platform charts. Istio and kube-prometheus-stack render at sync time from
  `bo-platform/platform/<env>/values/` — rendering them into git would be thousands of
  lines nobody reviews. You promote those by version bump, and the version bump *is*
  the diff.

## Non-negotiable (inherited from `bo-platform/CLAUDE.md`)

- **No floating tags. Ever.** Every image reference here is a **manifest-list index
  digest**. A per-arch digest pulls fine on one machine and fails `no match for platform`
  on the other — and this repo is what both clusters actually apply.
- Every rendered manifest carries the **commit-timestamp annotation** stamped in Phase 3.
  It is what makes DORA lead time measurable; retrofitting it is painful.
- **LF line endings**, enforced by `.gitattributes`.
- If reality contradicts the plan, **stop and say so.** Record it in
  `bo-platform/docs/DECISIONS.md`.

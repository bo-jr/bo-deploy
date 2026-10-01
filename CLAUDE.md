# bo-deploy — THE GATE

Rendered manifests only. **CI writes. Argo CD reads. Humans review and merge.**

Canonical spec: [`bo-platform/BUILD-PLAN.md`](https://github.com/bo-jr/bo-platform/blob/main/BUILD-PLAN.md)
§4 (why this repo is shared), Phase 3 (promotion), Phase 6 scenario 5 (atomicity). Where it and
[`bo-platform/DECISIONS.md`](https://github.com/bo-jr/bo-platform/blob/main/DECISIONS.md)
disagree, `DECISIONS.md` wins.

## Read this before touching anything

This repo is the **only manual gate in the entire system**. Merging a PR here deploys to
production. Everything else in the lab — the canary analysis, the burn-rate SLO, the
soak, the reconciler — exists to make the decision at this merge button an informed one.

```
rendered/
├── dev/<service>/manifests.yaml    <- written by GitHub Actions on a push to a service's main
└── prod/<service>/manifests.yaml   <- written by cmd/promoter after the soak passes
                                       (the promoter slice, after Phase 7)
```

Each `manifests.yaml` is byte for byte what `helm template` emits for the pinned chart
archive (`bo-platform/scripts/render-service.sh`, the lab's one render path). Argo CD's
`services` ApplicationSet in bo-platform turns each `rendered/<env>/<service>/` directory
into one Application.

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
- **0 required approving reviews** — not the 1 BUILD-PLAN specifies. `bo-jr` is the sole
  collaborator and GitHub does not let you approve your own PR, so 1 would make every
  merge an admin bypass: the bypass becomes the normal path and the gate decoration
  (DECISIONS 2026-09-12). Raise it when a second identity can approve — most likely
  `cmd/promoter` opening prod PRs under its own token.
- no force-push, no branch deletion
- dismiss stale approvals on new commits
- **required check `validate / rendered`** (Phase 3, below). Without a required check,
  `gh pr merge --auto` merges a PR the moment it opens, so auto-merge would gate nothing
  (DECISIONS 2026-09-30).
- **a `dev-health` status that re-queries dev at merge time** (the promoter slice, after
  Phase 7) — otherwise a PR opened Tuesday can be merged Thursday after dev has since
  degraded, and nothing catches it. This is the most commonly missed piece of a promotion
  pipeline. Hosted runners cannot reach the lab, so it is a commit status the in-cluster
  promoter refreshes on every run, not an Actions job.

`main-protection` is applied by `task repos:protect` from `bo-platform`, which loops one
ruleset payload over all seven via `gh api`. A personal account has no account-level
rulesets, so this is a seven-time setup. **`bo-deploy` is the one you would otherwise get
wrong**, and it alone also carries `deploy-checks` (`task repos:protect:deploy-checks`),
the ruleset that makes `validate` required.

## The `validate` check

`.github/workflows/validate.yml` calls `deploy-validate.yml` in bo-platform, pinned by
commit SHA. On every PR it requires:

1. only CI's `dev/<service>` branch changes `rendered/`, and only `rendered/dev/<service>/`
   — so "never hand-edit `rendered/`" is enforced, not just asked;
2. every rendered manifest passes the lab's Kyverno policies (digest-only images, no
   floating tags, explicit requests and limits);
3. every changed image resolves **anonymously** — how the clusters pull — as an OCI index
   of exactly `linux/amd64` + `linux/arm64`;
4. every changed image is cosign-signed by bo-platform's `service-ci.yml`, called by SHA,
   from that image's own service repo;
5. every workload carries a well-formed `gitops-lab/commit-timestamp`.

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

The dev PR's branch is `dev/<service>` and its title `dev: <service> <short sha>`. The PR
body carries the source commit, commit timestamp, index digest, chart digest, the run
link and copy-paste `cosign verify` / `gh attestation verify` commands — the only place
a SHA or digest appears outside `image:`. CI enables auto-merge (squash) and waits; if
the PR has not merged within the timeout, **the service's run fails**, on the repo that
is being watched. Promotions are serialized per service and never go backwards: a run
whose commit is no longer the tip of its `main` stands aside.

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
  digest**. CI renders on `linux/amd64` runners; every cluster that applies this repo is
  `arm64`. A per-arch digest passes CI and fails `no match for platform` in the lab.
- Every rendered workload carries **`gitops-lab/commit-timestamp`** (RFC3339, UTC) on its
  own metadata, stamped by the chart in Phase 3 — not on the pod template, Service or
  ServiceAccount. It is what makes DORA lead time measurable; retrofitting it is painful.
- **LF line endings**, enforced by `.gitattributes`.
- If reality contradicts the plan, **stop and say so.** Record it in
  `bo-platform/DECISIONS.md`.

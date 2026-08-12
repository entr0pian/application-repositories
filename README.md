# application-repositories

Source-of-truth repo for onboarding both services and cluster infrastructure into automated multi-cluster Argo CD delivery. A PR here — one small file — is the entire onboarding step; nothing needs to change in [`argocd`](https://github.com/entr0pian/argocd) itself.

Two structurally identical ApplicationSets there read directly from this repo:

- **`taskapp-catalog`** reads `catalog/<service>/<env>.yaml` — services (`backend`, `frontend`, ...).
- **`taskapp-infra`** reads `infra/<component>/<env>.yaml` — cluster infrastructure (`kube-prometheus-stack`, `platform`, `crossplane`, ...).

One file per service/component per environment, everything inline — no separate values tree for either. A `catalog/`+`values/` split was tried (the idea: scope a CI bot's write access to `values/` alone, away from `repoURL`/`chartPath`, via CODEOWNERS/branch protection) and reverted — nothing currently writes to either tree automatically. The old write-back path was retired along with `taskapp-argocd`'s `repositories-app.yaml`, and CI hasn't been repointed to write anywhere new yet (see "Not onboarded here" below). Building that boundary before its writer exists was speculative structure with no real benefit — worth re-introducing exactly when a high-frequency automated writer actually shows up, not before.

No CR, no operator, no write-back commit into another repo, ever. `<name>` and `<env>` always come from the file's own path, never from fields inside it. No cluster URL ever appears here either — both ApplicationSets resolve it live from ArgoCD's own registered clusters.

## catalog/ — services

```yaml
# catalog/backend/dev.yaml
repoURL: https://github.com/entr0pian/backend.git
targetRevision: main
chartPath: chart
namespace: default
values: |
  image:
    tag: "f84c7a80d3ac917285f7184a42e313fd357f8ee9"
```

| Field | Purpose |
|---|---|
| `repoURL` / `targetRevision` | The service's own repo and ref |
| `chartPath` | Where the chart lives inside that repo |
| `namespace` | Destination namespace |
| `values` | Raw Helm values YAML, inline — `""` if the chart's own defaults are fine |

## infra/ — cluster infrastructure

```yaml
# infra/platform/dev.yaml
repoURL: https://github.com/entr0pian/helm-charts.git
chart: ""
chartPath: platform
targetRevision: main
namespace: default
createNamespace: false
serverSideApply: false
wave: "1"
notify: true
values: |
  limitRange:
    enabled: true
  prometheusRules:
    enabled: true
  grafanaDashboard:
    enabled: true
  crossplane:
    secretPath: ""
  argocdWriteToken:
    secretPath: ""
  crossplaneGithub:
    secretPath: ""
```

| Field | Purpose |
|---|---|
| `repoURL` / `targetRevision` | Chart source |
| `chart` | Chart name — set this **or** `chartPath`, leave the other `""`. Use `chart` when `repoURL` is a Helm chart repository (e.g. `https://charts.crossplane.io/stable`) |
| `chartPath` | Path to the chart inside a git repo — use instead of `chart` when `repoURL` is a git repo (e.g. `helm-charts.git`) |
| `namespace` | Destination namespace |
| `createNamespace` / `serverSideApply` | Booleans → `CreateNamespace=`/`ServerSideApply=` sync options |
| `wave` | Sync-wave string, e.g. `"0"`, `"1"`, `"2"` |
| `notify` | Boolean — subscribe this Application to the `#deployments` Slack channel on sync succeeded/failed |
| `values` | Raw Helm values YAML, inline — `""` if the chart's own defaults are fine |

**Not onboarded here:** `crossplane-compositions-package` (an OCI `Configuration` CR, not a Helm chart at all) stays a hand-written Application template directly in `argocd` — confirmed structurally incompatible with the shared template above (`source.helm` and `source.directory` are both structs; unlike scalar fields, an unused one doesn't cleanly disappear from the rendered Application, so they can't safely coexist in one generic template). `backend-operator` isn't currently deployed anywhere (`operator.enabled` was `false` in every environment before this repo existed) — add `infra/backend-operator/<env>.yaml` the same way as `platform` above whenever it's actually needed. `backend`'s (and eventually `frontend`'s) CI still needs repointing to bump `catalog/<service>/<env>.yaml`'s `values.image.tag` on every push — the old mechanism that did this was retired along with `crs/`, and nothing has replaced it yet.

## To onboard

Add one file — `catalog/<service>/<env>.yaml` for a service, `infra/<component>/<env>.yaml` for infrastructure. Nothing to pair.

## History

This repo previously used `crs/` — `ApplicationRepository` custom resources reconciled by [`application-repository-operator`](https://github.com/entr0pian/application-repository-operator), which wrote generated deployment config back into `argocd`. That path has been retired: the operator's Argo CD Application was removed, and `crs/` was deleted. The operator's own repo still exists but is no longer deployed anywhere in this stack. Both `catalog/` and `infra/` briefly had a paired `values/` tree (split out for CI-write-scoping reasons) — folded back into one file each once it became clear nothing writes to either automatically yet, so the split had no real benefit to earn its keep against. Revisit per-tree, not globally, whenever an actual high-frequency writer shows up.

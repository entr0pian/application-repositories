# application-repositories

Source-of-truth repo for onboarding both services and cluster infrastructure into automated multi-cluster Argo CD delivery. A PR here — one or two small files — is the entire onboarding step; nothing needs to change in [`argocd`](https://github.com/entr0pian/argocd) itself.

Three structurally similar ApplicationSets there read directly from this repo:

- **`taskapp-catalog`** reads `components/<service>/environments/<env>.yaml` — services (`payments`, ...).
- **`taskapp-infra`** reads `infra/<component>/<env>.yaml` — cluster infrastructure (`kube-prometheus-stack`, `platform`, `crossplane`, ...).
- **`taskapp-packages`** reads `packages/<package>/<env>.yaml` — versioned Crossplane platform API packages (`crossplane-compositions`, ...).

`infra/` merges its identity file against a paired **`values/<name>/<env>.yaml`** via an ApplicationSet `merge` generator. `components/` keeps the same identity/values separation but as two sibling directories per component instead — `environments/<env>.yaml` (onboarding-time facts: repo, chart path, namespace) and `values/<env>.yaml` (what changes on every deploy: image tags, toggles) — read via a multi-source `Application` (`helm.valueFiles` pointing at the paired file by structural path, no merge generator). Both splits exist for the same reason: identity rarely changes and deserves human review, values change on every deploy and is the natural place to eventually scope a CI bot's write access via CODEOWNERS/branch protection, without that bot ever being able to touch where a chart lives. `packages/` doesn't follow either split — see its own section below for why.

No CR, no operator, no write-back commit into another repo, ever. `<name>` and `<env>` always come from the file's own path, never from fields inside it. No cluster URL ever appears here either — every ApplicationSet resolves it live from ArgoCD's own registered clusters.

## components/ — services

```yaml
# components/payments/environments/management.yaml
component: payments
environment: management
namespace: payments

source:
  repoURL: https://github.com/entr0pian/payments.git
  targetRevision: main
  chartPath: chart
```

| Field | Purpose |
|---|---|
| `component` | The component name — must match the directory (`components/<component>/...`) |
| `environment` | Used by `taskapp-catalog`'s cluster selector — must match the filename (`environments/<environment>.yaml`) |
| `namespace` | Destination namespace |
| `source.repoURL` / `source.targetRevision` | The service's own repo and ref |
| `source.chartPath` | Where the chart lives inside that repo |

```yaml
# components/payments/values/management.yaml — a real Helm values file, always required
# alongside environments/<env>.yaml (a missing pair is a sync error, not "no overrides")
image:
  tag: "41769607a44bfdab927e56d1ac89024e4b822cf3"
```

The generator glob (`components/*/environments/*.yaml`) structurally cannot match anything
under the sibling `components/<name>/values/` directory — the paired values file is
referenced by `helm.valueFiles` as a plain path convention, never re-globbed or merged in
as a second generator input. See [`RUNTIME_DEPENDENCIES.md`](https://github.com/entr0pian/platform-architecture/blob/main/RUNTIME_DEPENDENCIES.md#gitops-file-layout)
in `platform-architecture` for the full design and the ApplicationSet reshape it's built on.

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
values: ""
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
| `values` | Always `""` here — real overrides go in the paired `values/` file |

```yaml
# values/platform/dev.yaml
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

## packages/ — versioned platform API packages

```yaml
# packages/crossplane-compositions/management.yaml
name: crossplane-compositions-management
environment: management

source:
  repoURL: https://github.com/entr0pian/crossplane-compositions.git
  targetRevision: main
  path: charts/configuration-installer

package:
  repository: ghcr.io/entr0pian/crossplane-compositions
  version: v0.1.0

values:
  pullPolicy: IfNotPresent
```

| Field | Purpose |
|---|---|
| `environment` | Used by `taskapp-packages`' cluster selector, same as every other ApplicationSet here |
| `source.repoURL` / `source.targetRevision` / `source.path` | Where the installer Helm chart (`charts/configuration-installer` in the package's own repo) comes from — independent of the package version below |
| `package.repository` / `package.version` | Which immutable OCI `Configuration` artifact gets installed. `version` must always be an explicit release tag (e.g. `v0.4.1`) — never `latest`/`main`/`master` |
| `values` | Passed straight through to the installer chart (currently just `pullPolicy`) |

Unlike `catalog/`/`infra/`, this is a single file per package/environment — no paired
`values/` file. For those, identity rarely changes and values change on every deploy, so
splitting them keeps a CI bot's write access scopeable. For a package, `package.version`
— the one field that changes on every promotion — *is* the deployed identity; there's no
separate "identity" half to split it from. Promoting a package from dev to prod is a PR
that changes exactly this file's `package.version`. Publishing a new OCI version to GHCR
never deploys it anywhere by itself — only editing this file does.

Adding a new package (e.g. a future `platform-rds`) is just a new
`packages/<package>/<env>.yaml` file here — `argocd`'s `taskapp-packages` ApplicationSet
is generic over the contract above and needs no changes.

**Not onboarded here:** `backend`, `frontend`, and `backend-operator` aren't currently onboarded — `backend` was removed from `catalog/` as part of the `components/` reshape below and not yet re-added (add `components/backend/{environments,values}/<env>.yaml` the same way as `payments` whenever it's needed again); `backend-operator` needs `infra/backend-operator/<env>.yaml` (+ a paired `values/` file if needed) the same way as `platform` above. `payments`' (and eventually any other service's) CI still needs repointing to write `components/<service>/values/<env>.yaml`'s `image.tag` on every push — the old mechanism that did this was retired along with `crs/`, and nothing has replaced it yet (the planned `Release` API/controller, see `platform-architecture`, is meant to close this). Until that lands, `values/` is edited by hand like everything else here.

## To onboard

Add a `components/<service>/environments/<env>.yaml` + paired `components/<service>/values/<env>.yaml` pair for a service, or `infra/<component>/<env>.yaml` (+ paired `values/<name>/<env>.yaml` if needed) for infrastructure. For a versioned platform API package, add `packages/<package>/<env>.yaml` instead — see that section above for the contract shape.

## History

This repo previously used `crs/` — `ApplicationRepository` custom resources reconciled by [`application-repository-operator`](https://github.com/entr0pian/application-repository-operator), which wrote generated deployment config back into `argocd`. That path has been retired: the operator's Argo CD Application was removed, and `crs/` was deleted. The operator's own repo still exists but is no longer deployed anywhere in this stack. The `values/` split was briefly folded away (the reasoning: nothing writes to it automatically yet, so it wasn't earning its keep) and then restored — identity and parameters are a real separation of concerns worth keeping even before an automated writer exists for `values/`.

`catalog/<service>/<env>.yaml` (merged against `values/<service>/<env>.yaml` by an ApplicationSet `merge` generator, same shape as today's `infra/`) was retired in favor of `components/<service>/{environments,values}/<env>.yaml`, read via a multi-source `Application` (`helm.valueFiles`) instead — see the `components/` section above. The reshape avoids the `merge` generator's flat-key-lookup limitation (every file needed a redundant `name: <svc>-<env>` field purely to be a valid join key) and the `helm.values` string-templated-YAML-inside-YAML it relied on. `infra/`/`packages/` are unaffected — this reshape is scoped to `taskapp-catalog` only.

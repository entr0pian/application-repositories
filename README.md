# application-repositories

Source-of-truth repo for onboarding services into automated multi-cluster Argo CD delivery. A dev lead opens a PR here with one small file instead of the platform team hand-writing an Argo CD `Application` per repo per cluster.

A `taskapp-catalog` `ApplicationSet` in [`argocd`](https://github.com/entr0pian/argocd) reads `catalog/<service>/<env>.yaml` files here directly — no CR, no operator, no write-back commit into another repo.

## Layout

```
catalog/
└── <service>/
    └── <env>.yaml   # one file per service per environment
```

`<service>` and `<env>` come from the path itself, not from fields inside the file:

```yaml
# catalog/backend/dev.yaml
repoURL: https://github.com/entr0pian/backend.git
targetRevision: main
chartPath: chart
namespace: default
imageTag: "f84c7a80d3ac917285f7184a42e313fd357f8ee9"
```

No cluster URL here — the ApplicationSet resolves `dev`/`prod` to a real cluster by matching the `environment` label on ArgoCD's own registered clusters, so this file never needs to know what the current cluster URL happens to be.

To onboard a new service+environment, open a PR adding one file shaped like the one above.

No Helm chart, no templating — these are plain manifests read directly by the ApplicationSet's git-files generator, since each file only ever needs to describe one service+environment with no shared values to substitute.

## History

This repo previously used `crs/` — `ApplicationRepository` custom resources reconciled by [`application-repository-operator`](https://github.com/entr0pian/application-repository-operator), which wrote generated deployment config back into `argocd`. That path has been retired: the operator's Argo CD Application was removed, and `crs/` was deleted. The operator's own repo still exists but is no longer deployed anywhere in this stack.

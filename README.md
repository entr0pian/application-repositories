# application-repositories

Source-of-truth repo for `ApplicationRepository` custom resources — the onboarding mechanism for automated multi-cluster Argo CD delivery. A dev lead opens a PR here with a small CR instead of the platform team hand-writing an Argo CD `Application` per repo per cluster.

CRs committed here are synced by Argo CD into the management cluster, where the [`application-repository-operator`](https://github.com/entr0pian/application-repository-operator) reconciles them and writes the generated deployment config into [`argocd`](https://github.com/entr0pian/argocd).

## Layout

```
crs/
└── <repo-name>.yaml   # one ApplicationRepository CR per onboarded repo
```

To onboard a new repo, open a PR adding `crs/<repo-name>.yaml`:

```yaml
apiVersion: platform.taskapp.io/v1alpha1
kind: ApplicationRepository
metadata:
  name: <repo-name>
  namespace: default
spec:
  repoURL: https://github.com/entr0pian/<repo-name>.git
  targetRevision: main
  chartPath: chart
  clusters:
    - name: dev
      namespace: default
```

No Helm chart, no templating — these are plain manifests applied via an Argo CD `directory` source, since each file only ever needs to describe one CR with no shared values to substitute.

# application-repositories

Source-of-truth repo for `ApplicationRepository` custom resources — the onboarding mechanism for automated multi-cluster Argo CD delivery. A dev lead opens a PR here with a small CR instead of the platform team hand-writing an Argo CD `Application` per repo per cluster.

CRs committed here are synced by Argo CD into the management cluster, where the [`application-repository-operator`](https://github.com/entr0pian/application-repository-operator) reconciles them and writes the generated deployment config into [`argocd`](https://github.com/entr0pian/argocd).

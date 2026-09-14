# argocd-gitops

Argo CD **App of AppSets** repository: a single `root` Application bootstraps an
`ApplicationSet`, which fans out every app to every cluster from Git descriptors.

## Structure

```
.
├── bootstrap/
│   ├── root-app.yaml          # root Application -> bootstrap/apps (app-of-apps)
│   └── apps/
│       ├── project.yaml       # AppProject "platform" (scoped)
│       └── platform-appset.yaml # ApplicationSet (matrix: apps x clusters)
├── clusters/
│   ├── dev/config.yaml        # cluster metadata (name, environment, server)
│   └── prod/config.yaml
└── apps/
    ├── cert-manager/
    │   ├── application.yaml   # app descriptor (chart, version, namespace, wave)
    │   ├── values.yaml        # common Helm values
    │   ├── values-dev.yaml    # dev overrides
    │   └── values-prod.yaml   # prod overrides
    ├── ingress-nginx/  (same layout)
    ├── monitoring/     (same layout)
    └── argo-cd/        (same layout)
```

## How it works

1. **Install Argo CD (once, manual):** the `Application` CRD only exists
   after Argo CD is installed.

   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts
   kubectl -n argocd wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server --timeout=300s
   ```

2. **Bootstrap (once, manual):**

   ```bash
   kubectl apply -f bootstrap/root-app.yaml
   ```

3. **App of Apps** — `root` (a plain directory `Application`) syncs
   `bootstrap/apps/`, creating the `platform` `AppProject` and the `platform`
   `ApplicationSet`.

4. **App of AppSets** — the `ApplicationSet` uses a `matrix` of two `git`
   generators over `apps/*/application.yaml` (one app per file) and
   `clusters/*/config.yaml` (one cluster per file). For every (app, cluster)
   pair it generates an `Application` named `<app>-<cluster>`.

5. **Helm multi-source** — each generated `Application` pulls the Helm chart from
   its Helm repo and the values from this repo via the `$values` ref:
   `values.yaml` (common) + `values-<environment>.yaml` (cluster override).

## Apps

| App | Chart | Namespace | Wave |
| --- | --- | --- | --- |
| cert-manager | `charts.jetstack.io/cert-manager` | `cert-manager` | 0 |
| ingress-nginx | `kubernetes.github.io/ingress-nginx` | `ingress-nginx` | 1 |
| monitoring | `prometheus-community/kube-prometheus-stack` | `monitoring` | 2 |
| argo-cd | `argoproj.github.io/argo-helm` | `argocd` | 3 |

`cert-manager` syncs first (wave 0) so its CRDs exist before `ingress-nginx` and
`monitoring` consume them. `argo-cd` runs last and manages Argo CD itself —
complete the self-management loop by making sure the chart version and values
match how you originally installed Argo CD, and review changes before syncing to
avoid locking yourself out.

## Clusters

`clusters/*/config.yaml` are the source of truth consumed by the `ApplicationSet`
git generator. The generated `Application.destination.server` must still be a
cluster **registered in Argo CD**. `dev` targets the in-cluster
`https://kubernetes.default.svc`; register `prod` (the placeholder
`https://production-cluster.example.com`) with `argocd cluster add` or a cluster
Secret before its apps can sync.

## Conventions

- **Scoped project** — apps run in the `platform` `AppProject`, whose
  `sourceRepos`/`destinations` are allow-listed (not the wide-open `default`).
- **Pinned versions** — chart versions are pinned; the git generator tracks
  `main`. Pin `revision` to a tag/SHA for production.
- **Ignore differences** — the root app ignores drift on `ApplicationSet`
  refresh metadata; generated apps ignore `Deployment.spec.replicas` (leave room
  for HPA/manual scaling).
- **Sync waves** — `argocd.argoproj.io/sync-wave` orders installation.

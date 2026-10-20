# infra-argocd-datamatiker

GitOps repo for a local minikube cluster, managed by Argo CD. Any machine can
rebuild the same cluster from this repo.

## Bootstrap

```bash
./bootstrap/install.sh
```

This starts minikube (if it isn't running), installs Argo CD with Helm from
`charts/argocd`, and applies the root ApplicationSet. It prints the admin
password and how to open the UI when done.

Override minikube resources with `MINIKUBE_CPUS` / `MINIKUBE_MEMORY` (MB).

## Self-managed Argo CD

`apps/argocd` deploys `charts/argocd` with release name `argocd` — the same chart
and values `install.sh` installs — so Argo CD adopts its own install. To change
Argo CD (version or config), edit `charts/argocd` and push. It syncs
automatically but never prunes, and deleting the Application does not delete
Argo CD.

## Layout

```
bootstrap/
  install.sh           one-time setup of a fresh cluster
  root-appset.yaml     root ApplicationSet: one Application per apps/<svc>/
apps/<svc>/            Argo CD Application for the service (application.yaml)
charts/<svc>/          the service's Helm chart (umbrella chart + values.yaml)
```

## Adding an app

1. Create `charts/<svc>/` with `Chart.yaml` (upstream chart under
   `dependencies:`) and `values.yaml`.
2. Create `apps/<svc>/application.yaml` pointing at `charts/<svc>`.
3. Push to `main` — Argo CD picks it up automatically.

Example `apps/<svc>/application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <svc>
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/MikkelRLarsen/infra-argocd-datamatiker.git
    targetRevision: main
    path: charts/<svc>
    helm:
      releaseName: <svc>
      valueFiles: [values.yaml]
  destination:
    server: https://kubernetes.default.svc
    namespace: <svc>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

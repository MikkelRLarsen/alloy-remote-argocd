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



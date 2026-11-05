# cicdpipeline_deployment

GitOps deployment repo for **cicd-app**. Argo CD (running in EKS) watches this repo and keeps the cluster in sync with it.

```
k8s/
  kustomization.yaml   <- Jenkins updates images[].newTag here on every build
  deployment.yaml      <- 2 replicas, probes on /health, non-root, read-only filesystem
  service.yaml         <- LoadBalancer (public URL)
  pdb.yaml             <- keeps at least 1 pod up during node maintenance
argocd/
  application.yaml     <- Argo CD Application (auto-sync, prune, self-heal)
```

## Flow

1. Code is pushed to [cicdpipeline_project](https://github.com/aravindrajsaravanan/cicdpipeline_project).
2. Jenkins tests, scans and pushes the image to ECR, then commits the new tag to `k8s/kustomization.yaml` here.
3. Argo CD notices the commit (polls every ~3 min) and rolls out the new image.

## Install Argo CD (once per cluster)

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.6.1/manifests/install.yaml
kubectl -n argocd rollout status deploy --timeout=300s
kubectl apply -f argocd/application.yaml
```

### Open the Argo CD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8443:443
```

Browse to https://localhost:8443 (self-signed certificate warning is expected). User: `admin`.
Initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Change it after the first login (User Info -> Update Password).

## Rules

- Don't edit the cluster by hand. Argo CD's self-heal reverts manual changes. Change this repo instead.
- Rollback = `git revert` the Jenkins commit that bumped the tag.

## Preview locally

```bash
kubectl kustomize k8s/
```

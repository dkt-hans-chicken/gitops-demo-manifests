# gitops-demo-manifests

Kubernetes manifests for gitops-demo. ArgoCD watches this repo and syncs to the cluster.

## Structure

```
k8s/
└── staging/
    ├── namespace.yaml     # staging namespace
    ├── deployment.yaml    # app deployment (image tag updated by CI)
    └── service.yaml       # ClusterIP service
argocd-app.yaml            # ArgoCD Application definition
```

## Setup ArgoCD on test-cluster

### 1. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl get pods -n argocd -w
```

### 2. Access ArgoCD UI

```bash
# Port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Login at https://localhost:8080
# Username: admin
# Password: (from above)
```

### 3. Apply the ArgoCD Application

```bash
# Replace <YOUR_ORG> in argocd-app.yaml first
kubectl apply -f argocd-app.yaml

# ArgoCD will now watch k8s/staging/ and sync automatically
```

### 4. Verify sync

```bash
# Install argocd CLI
brew install argocd

# Login
argocd login localhost:8080 --username admin --insecure

# Check app status
argocd app get gitops-demo-staging
argocd app list
```

## How image tag gets updated

CI in `gitops-demo-app` repo runs `sed` on `k8s/staging/deployment.yaml`:

```bash
sed -i "s|image: .*/gitops-demo:.*|image: your-user/gitops-demo:abc1234|g" \
  k8s/staging/deployment.yaml
git commit -m "ci: update staging image to abc1234"
git push
```

ArgoCD detects the commit within ~3 minutes and syncs automatically.

## Manual sync (if needed)

```bash
argocd app sync gitops-demo-staging
```

## Rollback

```bash
# Option 1 — git revert (preferred GitOps way)
git revert HEAD
git push

# Option 2 — ArgoCD UI: History → rollback to previous version

# Option 3 — argocd CLI
argocd app rollback gitops-demo-staging
```

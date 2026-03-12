# Manual Install Configuration

This directory contains `values.yaml` files for components that require manual installation during the initial cluster bootstrap.

## Structure

```
infra/.manual-install/
├── calico/
│   └── values.yaml    # CNI Configuration
└── argo-cd/
    └── values.yaml    # Argo CD Configuration
```

## Installation Commands

### 1. Calico (CNI)

```bash
cd infra/.manual-install/calico

helm repo add projectcalico https://projectcalico.docs.tigera.io/charts
helm repo update

helm upgrade --install calico projectcalico/tigera-operator \
  --version 3.31.2 \
  -n tigera-operator --create-namespace \
  -f values.yaml
```

### 2. Argo CD

```bash
cd infra/.manual-install/argo-cd

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd \
  --version 9.1.7 \
  -n argo-cd --create-namespace \
  -f values.yaml
```

**Check Initial Admin Password:**

```bash
kubectl -n argo-cd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 2-1. Add Git Repository to Argo CD

After Argo CD is installed, add the Git repository using `infra/argo-cd/repository.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: k8s-operations
  namespace: argo-cd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  name: k8s-operations
  url: https://github.com/hosytt/k8s-operations.git
  insecure: "true"
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    -----END OPENSSH PRIVATE KEY-----
```

Apply:

```bash
kubectl apply -f infra/argo-cd/repository.yaml
```

## Installation Order

During the bootstrap phase, please install in the following order:

1. **Calico** (Required for Networking)
2. **Argo CD** (Required for GitOps)

Once Argo CD is installed, other components will be deployed automatically via GitOps.

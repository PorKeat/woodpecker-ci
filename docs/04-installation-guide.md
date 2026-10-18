# 04 — Installation & Deployment Guide

This guide contains the step-by-step instructions to deploy both **ArgoCD** and **Woodpecker CI** on **`Node1`** (`10.1.16.11`).

---

## Prerequisites
- Kubernetes Cluster: `Node1` (`10.1.16.11`), `Node2` (`10.1.16.12`), `Node3` (`10.1.16.13`)
- LoadBalancer VIP: `10.1.16.200` (Traefik Ingress Controller)
- StorageClass: `longhorn`
- Vault Server: `10.1.18.8` (Vault 2.1.0)
- Woodpecker Domain: `https://woodpecker.unity-workspace.com/`

---

## STEP 1: Deploy ArgoCD (Continuous Deployment)

Log in to **`Node1`** via SSH:
```bash
ssh Node1
```

### 1.1 Create Namespace & TLS Secret
```bash
kubectl create namespace argocd

# Generate self-signed TLS certificate with Subject Alternative Names (SAN)
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout /tmp/argocd.key -out /tmp/argocd.crt \
  -subj "/CN=argocd" \
  -addext "subjectAltName=IP:10.1.16.200,DNS:argocd,DNS:localhost"

kubectl create secret tls argocd-tls \
  --cert=/tmp/argocd.crt \
  --key=/tmp/argocd.key \
  -n argocd

rm -f /tmp/argocd.key /tmp/argocd.crt
```

### 1.2 Create VPN-Only Middleware (Traefik IP Allowlist)
```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: argocd-vpn-only
  namespace: argocd
spec:
  ipAllowList:
    sourceRange:
      - 10.0.0.0/8
      - 100.64.0.0/10
      - 172.16.0.0/12
      - 192.168.0.0/16
      - 127.0.0.1/32
EOF
```

### 1.3 Install ArgoCD via Helm
Use the manifest from [`manifests/argocd/values.yaml`](../manifests/argocd/values.yaml):

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argo argo/argo-cd -n argocd -f manifests/argocd/values.yaml
```

### 1.4 Apply Ingress Manifests
```bash
kubectl apply -f manifests/argocd/ingress.yaml
```

---

## STEP 2: Deploy Woodpecker CI (Continuous Integration)

### 2.1 Create Namespace & VPN Middleware
```bash
kubectl create namespace woodpecker

cat << 'EOF' | kubectl apply -f -
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: woodpecker-vpn-only
  namespace: woodpecker
spec:
  ipAllowList:
    sourceRange:
      - 10.0.0.0/8
      - 100.64.0.0/10
      - 172.16.0.0/12
      - 192.168.0.0/16
      - 127.0.0.1/32
EOF
```

### 2.2 Install Woodpecker via Helm
Use the production manifest from [`manifests/woodpecker/values.yaml`](../manifests/woodpecker/values.yaml):

```bash
helm repo add woodpecker https://woodpecker-ci.org/
helm repo update
helm upgrade --install woodpecker woodpecker/woodpecker \
  --namespace woodpecker \
  -f manifests/woodpecker/values.yaml
```

### 2.3 Apply Traefik Ingress
Apply the ingress from [`manifests/woodpecker/ingress.yaml`](../manifests/woodpecker/ingress.yaml):
```bash
kubectl apply -f manifests/woodpecker/ingress.yaml
```

### 2.4 Verify
Open **[https://woodpecker.unity-workspace.com/](https://woodpecker.unity-workspace.com/)** and log in with GitHub.
See [`docs/06-user-and-admin-management.md`](06-user-and-admin-management.md) for team and admin configuration.

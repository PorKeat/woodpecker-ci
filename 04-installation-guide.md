# 04 — Installation & Deployment Guide

This guide contains the complete, step-by-step instructions to install and configure both **ArgoCD** and **Woodpecker CI** on **`Node1`** (`10.1.16.11`).

---

## Prerequisites Check
- Kubernetes Cluster: `Node1` (`10.1.16.11`), `Node2` (`10.1.16.12`), `Node3` (`10.1.16.13`)
- LoadBalancer IP: `10.1.16.200` (Traefik Ingress Controller)
- StorageClass: `longhorn`
- Vault Server: `10.1.18.8` (Vault 2.1.0)
- GitHub Username: `PorKeat`

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

### 1.3 Create Helm Values (`~/argocd-values.yaml`)
Notice that the bcrypt password hash from Vault is pre-populated here so ArgoCD uses the real Vault password immediately:

```bash
cat << 'EOF' > ~/argocd-values.yaml
global:
  image:
    repository: quay.io/argoproj/argocd
    tag: "v3.5.5"

configs:
  params:
    server.insecure: true
  cm:
    url: "https://10.1.16.200"
    admin.enabled: "true"
  secret:
    # Pre-computed bcrypt hash of: ArgoAdmin2026SecOps!
    argocdServerAdminPassword: "$2b$12$pIVPmWwcD4bX6DET4PLzrOsB5S/yEfnhQUW5PAbaFmF8ON5iJTHHO"

server:
  replicas: 1
  ingress:
    enabled: false

controller:
  replicas: 1

repoServer:
  replicas: 1

redis:
  enabled: true
  image:
    repository: ecr-public.aws.com/docker/library/redis
    tag: "8.6.4-alpine"

dex:
  enabled: false

notifications:
  enabled: false
EOF
```

### 1.4 Install ArgoCD via Helm
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argo argo/argo-cd -n argocd -f ~/argocd-values.yaml
```

### 1.5 Create ArgoCD Ingress
```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: "web,websecure"
    traefik.ingress.kubernetes.io/router.middlewares: "argocd-argocd-vpn-only@kubernetescrd"
spec:
  ingressClassName: traefik
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argo-argocd-server
            port:
              number: 80
  tls:
  - secretName: argocd-tls
EOF
```

### 1.6 Verify ArgoCD
1. Open [https://10.1.16.200](https://10.1.16.200) in your browser while connected to VPN.
2. Sign in with:
   - **Username:** `admin`
   - **Password:** `ArgoAdmin2026SecOps!`

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

### 2.2 Create Helm Values (`~/woodpecker-values.yaml`)
Replace `<YOUR_CLIENT_ID>` and `<YOUR_CLIENT_SECRET>` with the values from your GitHub OAuth App:

```bash
cat << 'EOF' > ~/woodpecker-values.yaml
server:
  enabled: true
  replicaCount: 1
  image:
    repository: woodpeckerci/woodpecker-server
    tag: "v3.19.0"
  env:
    WOODPECKER_HOST: "https://10.1.16.200/woodpecker"
    WOODPECKER_SERVER_ADDR: ":8000"
    WOODPECKER_SERVER_RPC_ADDR: ":9000"
    WOODPECKER_ROOT_URL: "/woodpecker"
    # Vault RPC secret:
    WOODPECKER_SERVER_RPC_SECRET: "c9ecc71b30612bfc502165628a06d9d7612663536d9662ab4e0b2dc839356f43"
    WOODPECKER_ADMIN: "PorKeat"
    WOODPECKER_GITHUB: "true"
    WOODPECKER_GITHUB_CLIENT: "<YOUR_CLIENT_ID>"
    WOODPECKER_GITHUB_SECRET: "<YOUR_CLIENT_SECRET>"

agent:
  enabled: true
  replicaCount: 1
  image:
    repository: woodpeckerci/woodpecker-agent
    tag: "v3.19.0"
  env:
    WOODPECKER_SERVER: "woodpecker-server:9000"
    WOODPECKER_SERVER_RPC_SECRET: "c9ecc71b30612bfc502165628a06d9d7612663536d9662ab4e0b2dc839356f43"
    WOODPECKER_BACKEND: "kubernetes"
    WOODPECKER_BACKEND_K8S_NAMESPACE: "woodpecker"
    WOODPECKER_BACKEND_K8S_STORAGE_CLASS: "longhorn"
    WOODPECKER_BACKEND_K8S_VOLUME_SIZE: "10G"
EOF
```

### 2.3 Install Woodpecker via Helm
```bash
helm repo add woodpecker https://woodpecker-ci.org/
helm repo update
helm upgrade --install woodpecker woodpecker/woodpecker \
  --namespace woodpecker \
  -f ~/woodpecker-values.yaml
```

### 2.4 Create Woodpecker Ingress
```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: woodpecker-ingress
  namespace: woodpecker
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: "web,websecure"
    traefik.ingress.kubernetes.io/router.middlewares: "woodpecker-woodpecker-vpn-only@kubernetescrd"
spec:
  ingressClassName: traefik
  rules:
  - http:
      paths:
      - path: /woodpecker
        pathType: Prefix
        backend:
          service:
            name: woodpecker-server
            port:
              number: 8000
  tls:
  - secretName: argocd-tls
EOF
```

### 2.5 Verify Woodpecker
1. Open [https://10.1.16.200/woodpecker](https://10.1.16.200/woodpecker) in your browser.
2. Click **Login with GitHub** and authorize your account (`PorKeat`).
3. You will see all your GitHub repositories listed ready to enable pipelines!

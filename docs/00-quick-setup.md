# ⚡ Quick Setup Guide (In 3 Minutes)

This is the fastest, copy-paste guide to set up Woodpecker CI and ArgoCD on Kubernetes.

---

## Step 1: Create GitHub OAuth App (30 seconds)

1. Open: **[https://github.com/settings/applications/new](https://github.com/settings/applications/new)**
2. Fill in:
   * **Application name**: `Woodpecker CI`
   * **Homepage URL**: `https://woodpecker.unity-workspace.com`
   * **Authorization callback URL**: `https://woodpecker.unity-workspace.com/authorize`
3. Click **Register application**.
4. Copy your **Client ID** and generate a **Client Secret**.

---

## Step 2: Save Credentials to Vault (10 seconds)

SSH into `Node1` or your Vault machine:

```bash
vault kv patch secret/woodpecker \
  github_client_id="<YOUR_CLIENT_ID>" \
  github_client_secret="<YOUR_CLIENT_SECRET>"
```

---

## Step 3: Deploy ArgoCD (1 minute)

SSH into `Node1`:

```bash
# 1. Apply namespace & VPN allowlist
kubectl apply -f manifests/argocd/01-namespace.yaml
kubectl apply -f manifests/argocd/02-vpn-middleware.yaml

# 2. Install ArgoCD via Helm
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argo argo/argo-cd -n argocd -f manifests/argocd/03-helm-values.yaml

# 3. Apply Ingress
kubectl apply -f manifests/argocd/04-ingress.yaml
```

* **ArgoCD Dashboard**: `https://10.1.16.200/argocd`
* **Username**: `admin`
* **Password**: `SJh7RxGCn0EAz5FRvSWLC_dYY-AA`

---

## Step 4: Deploy Woodpecker CI (1 minute)

```bash
# 1. Apply namespace & VPN allowlist
kubectl apply -f manifests/woodpecker/01-namespace.yaml
kubectl apply -f manifests/woodpecker/02-vpn-middleware.yaml

# 2. Install Woodpecker via Helm
helm repo add woodpecker https://woodpecker-ci.org/
helm repo update
helm upgrade --install woodpecker woodpecker/woodpecker -n woodpecker -f manifests/woodpecker/03-helm-values.yaml

# 3. Apply Ingress
kubectl apply -f manifests/woodpecker/04-ingress.yaml
```

* **Woodpecker Dashboard**: [https://woodpecker.unity-workspace.com/](https://woodpecker.unity-workspace.com/)

---

## Step 5: Log in & Team Management

1. Open [https://woodpecker.unity-workspace.com/](https://woodpecker.unity-workspace.com/).
2. Click **Login with GitHub** (your account `PorKeat` is the initial superadmin).
3. All members of `unity-workspace-org` can log in with their own accounts.
4. To make a teammate an admin:
   * Go to **Settings** / **Administration** $\rightarrow$ **Users**.
   * Turn **Admin** switch ON for that teammate. Done!

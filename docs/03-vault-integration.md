# 03 — HashiCorp Vault Integration

The CI/CD platform is 100% **Vault-Native**. No secrets, passwords, RPC tokens, or OAuth credentials are saved in Kubernetes `values.yaml` files, ConfigMaps, or Git repositories.

All secrets are maintained in **HashiCorp Vault** on `10.1.18.8` and dynamically injected into running pods at boot time via the **Vault Agent Sidecar Injector** (`vault-system`).

---

## 1. Vault Secret Hierarchy

All CI/CD secrets reside in the Version 2 KV secrets engine under `secret/data/`:

```
secret/
  ├── argocd/
  │     ├── url: "https://10.1.16.200/argocd"
  │     ├── admin_user: "admin"
  │     ├── admin_password: "SJh7RxGCn0EAz5FRvSWLC_dYY-AA"
  │     ├── admin_password_bcrypt: "$2a$10$iOH60jL/wI.sukHDt117o.MGLMATFPzjdUCfVTbqCBScrMU9M7PKi"
  │     └── access: "vpn-only"
  │
  └── woodpecker/
        ├── url: "https://10.1.16.200/woodpecker"
        ├── admin_user: "PorKeat"
        ├── rpc_secret: "c9ecc71b30612bfc502165628a06d9d7612663536d9662ab4e0b2dc839356f43"
        ├── github_client_id: "Ov23lia9YFRFXGSycBne"
        ├── github_client_secret: "a3700e5a07f31ee898f9c050d6798d7abcfc7c46"
        └── access: "vpn-only"
```

---

## 2. Vault Security Policies & Roles (Configured on 10.1.18.8)

### Woodpecker CI Policy & Role
```hcl
# Policy: woodpecker-policy
path "secret/data/woodpecker" {
  capabilities = ["read"]
}
```
Bound to ServiceAccounts `woodpecker-agent`, `default`, `woodpecker-server` in namespace `woodpecker`:
```bash
vault write auth/kubernetes/role/woodpecker-role \
  bound_service_account_names="woodpecker-agent,default,woodpecker-server" \
  bound_service_account_namespaces="woodpecker" \
  policies="woodpecker-policy" \
  audience="https://kubernetes.default.svc.cluster.local" \
  ttl="24h"
```

### ArgoCD Policy & Role
```hcl
# Policy: argocd-policy
path "secret/data/argocd" {
  capabilities = ["read"]
}
```
Bound to ServiceAccount `argocd-server` in namespace `argocd`:
```bash
vault write auth/kubernetes/role/argocd-role \
  bound_service_account_names="argocd-server,default,argo-argocd-server" \
  bound_service_account_namespaces="argocd" \
  policies="argocd-policy" \
  audience="https://kubernetes.default.svc.cluster.local" \
  ttl="24h"
```

---

## 3. How the Vault Agent Sidecar Works in Kubernetes

Each pod is annotated with Vault Agent annotations. When Kubernetes schedules the pod:

1. The **Vault Agent Injector** (`vault-system`) injects two containers:
   - `vault-agent-init` (initContainer)
   - `vault-agent` (sidecar container)
2. The sidecar securely contacts `https://10.1.18.8:8200` using the pod's projected Kubernetes ServiceAccount token (`/var/run/secrets/kubernetes.io/serviceaccount/token`).
3. Vault verifies the token against the Kubernetes API, checks the policy, and issues a leased token.
4. The sidecar renders an in-memory `tmpfs` volume (`/vault/secrets/`) containing the decrypted secrets.
5. In-memory secrets are never persisted to disk or etcd.

---

## 4. Pod Annotations Reference

### Woodpecker CI Server Annotations:
```yaml
podAnnotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/agent-vault-addr: "https://10.1.18.8:8200"
  vault.hashicorp.com/tls-skip-verify: "true"
  vault.hashicorp.com/role: "woodpecker-role"
  vault.hashicorp.com/agent-inject-secret-woodpecker.env: "secret/data/woodpecker"
  vault.hashicorp.com/agent-inject-template-woodpecker.env: |
    {{- with secret "secret/data/woodpecker" -}}
    export WOODPECKER_SERVER_RPC_SECRET="{{ .Data.data.rpc_secret }}"
    export WOODPECKER_GITHUB_CLIENT="{{ .Data.data.github_client_id }}"
    export WOODPECKER_GITHUB_SECRET="{{ .Data.data.github_client_secret }}"
    {{- end -}}
```

### ArgoCD Server Annotations:
```yaml
podAnnotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/agent-vault-addr: "https://10.1.18.8:8200"
  vault.hashicorp.com/tls-skip-verify: "true"
  vault.hashicorp.com/role: "argocd-role"
  vault.hashicorp.com/agent-inject-secret-argocd.env: "secret/data/argocd"
  vault.hashicorp.com/agent-inject-template-argocd.env: |
    {{- with secret "secret/data/argocd" -}}
    export ARGOCD_ADMIN_PASSWORD="{{ .Data.data.admin_password }}"
    export ARGOCD_ADMIN_PASSWORD_BCRYPT="{{ .Data.data.admin_password_bcrypt }}"
    {{- end -}}
```

---

## 5. Verifying Vault Secrets & Sidecars

### Inspect Injected Secret in Running Woodpecker Pod:
```bash
kubectl exec -n woodpecker woodpecker-server-0 -c vault-agent -- cat /vault/secrets/woodpecker.env
```

### Inspect Injected Secret in Running ArgoCD Pod:
```bash
kubectl exec -n argocd deploy/argo-argocd-server -c vault-agent -- cat /vault/secrets/argocd.env
```

### Rotate Secrets in Vault:
Whenever you update a secret in Vault:
```bash
vault kv patch secret/woodpecker ...
vault kv patch secret/argocd ...
```
The Vault Agent sidecar automatically detects the secret change in Vault and refreshes the in-memory `/vault/secrets/` file in real-time without restarting the pod.

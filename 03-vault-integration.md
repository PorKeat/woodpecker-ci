# 03 — HashiCorp Vault Integration

Your infrastructure includes a dedicated HashiCorp Vault cluster running on VM `10.1.18.8` (Vault 2.1.0, Shamir 5-shares/threshold-3 unsealed). The Kubernetes cluster (`Node1-3`) runs the **Vault Agent Injector** (`vault-system` namespace), which allows pods to read secrets directly from Vault without hardcoding them in Git.

---

## 1. Vault Secret Hierarchy

All CI/CD credentials are stored in the Version 2 KV secrets engine at `secret/data/`:

```
secret/
  ├── argocd/
  │     ├── url: "https://10.1.16.200"
  │     ├── admin_user: "admin"
  │     ├── admin_password: "ArgoAdmin2026SecOps!"
  │     ├── admin_password_bcrypt: "$2b$12$pIVPmWwcD4bX6DET4PLzrOsB5S/yEfnhQUW5PAbaFmF8ON5iJTHHO"
  │     └── access: "vpn-only"
  │
  └── woodpecker/
        ├── url: "https://10.1.16.200/woodpecker"
        ├── admin_user: "PorKeat"
        ├── rpc_secret: "c9ecc71b30612bfc502165628a06d9d7612663536d9662ab4e0b2dc839356f43"
        ├── github_client_id: "<your_client_id>"
        ├── github_client_secret: "<your_client_secret>"
        └── access: "vpn-only"
```

---

## 2. Managing Secrets via Vault CLI

Run these commands on the **Vault VM** (`ssh Vault` or `10.1.18.8`):

### View Stored ArgoCD Credentials
```bash
vault kv get secret/argocd
```

### View Stored Woodpecker Credentials
```bash
vault kv get secret/woodpecker
```

### Update / Patch Credentials
```bash
# Update Woodpecker's GitHub OAuth credentials
vault kv patch secret/woodpecker \
  github_client_id="<NEW_CLIENT_ID>" \
  github_client_secret="<NEW_CLIENT_SECRET>"

# Update ArgoCD admin password
vault kv patch secret/argocd \
  admin_password="<NEW_PASSWORD>" \
  admin_password_bcrypt="<NEW_BCRYPT_HASH>"
```

### Read a Single Field Programmatically
```bash
# Fetch just the ArgoCD admin password
ARGOCD_PASS=$(vault kv get -field=admin_password secret/argocd)

# Fetch just the Woodpecker RPC secret
RPC_SECRET=$(vault kv get -field=rpc_secret secret/woodpecker)
```

---

## 3. Dynamic Secret Injection into Kubernetes Pods

Because the **Vault Agent Injector** is active in the cluster, any Kubernetes deployment can dynamically mount secrets into `/vault/secrets/` as an in-memory volume.

### Example: Injecting Vault Secrets into a Pod
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: default
spec:
  template:
    metadata:
      annotations:
        # Enable the Vault agent sidecar injector
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/role: 'kubernetes'
        # Read from Vault secret/data/woodpecker
        vault.hashicorp.com/agent-inject-secret-config: 'secret/data/woodpecker'
        # Format the output inside the pod
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "secret/data/woodpecker" -}}
          export WOODPECKER_RPC_SECRET="{{ .Data.data.rpc_secret }}"
          export WOODPECKER_ADMIN="{{ .Data.data.admin_user }}"
          {{- end -}}
    spec:
      containers:
        - name: app
          image: my-image:latest
          command: ["/bin/sh", "-c"]
          args: ["source /vault/secrets/config && ./start-service"]
```

### Benefits:
- **Zero Plaintext Secrets in Git:** Manifests stored in Git only contain the annotation pointer `secret/data/...`.
- **Automatic Secret Rotation:** If a secret is updated in Vault, the injector refreshes the mounted file inside the pod automatically.
- **Auditing:** Vault maintains an audit log of every token, pod, and service account that accessed secrets.

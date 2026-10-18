# 🚀 Woodpecker CI + ArgoCD GitOps

A lightweight, secure CI/CD platform running natively on Kubernetes, integrated with HashiCorp Vault, and accessible at **[https://woodpecker.unity-workspace.com/](https://woodpecker.unity-workspace.com/)**.

---

## ⚡ Quick Start
Want to deploy in 3 minutes? Follow the **[Quick Setup Guide](docs/00-quick-setup.md)**.

---

## 📁 Repository Structure

```text
woodpecker-ci/
├── docs/                                 # 📖 Documentation Guides
│   ├── 00-quick-setup.md                 # ⚡ 3-Minute setup guide
│   ├── 01-architecture-and-concepts.md   # Architectural design & how it works
│   ├── 02-github-oauth-setup.md          # GitHub OAuth setup guide
│   ├── 03-vault-integration.md           # HashiCorp Vault secrets integration
│   ├── 04-installation-guide.md          # Full manual installation guide
│   ├── 05-pipeline-syntax-guide.md       # .woodpecker.yaml syntax & Kaniko builds
│   └── 06-user-and-admin-management.md   # 👤 WOODPECKER_ADMIN, ORGS & team permissions
│
├── manifests/                            # 📦 Production Kubernetes Manifests
│   ├── woodpecker/                       # Woodpecker CI Server & Agent
│   │   ├── 01-namespace.yaml             # Namespace 'woodpecker'
│   │   ├── 02-vpn-middleware.yaml        # Traefik IP allowlist
│   │   ├── 03-service.yaml               # ClusterIP & Headless Services
│   │   ├── 04-deployment.yaml            # Server & Agent StatefulSets
│   │   ├── 05-ingress.yaml               # Traefik Ingress (root / and /woodpecker)
│   │   └── helm-values.yaml              # Production Helm values alternative
│   │
│   ├── argocd/                           # ArgoCD GitOps Engine
│   │   ├── 01-namespace.yaml             # Namespace 'argocd'
│   │   ├── 02-vpn-middleware.yaml        # Traefik IP allowlist
│   │   ├── 03-helm-values.yaml           # Pinned ArgoCD Helm values
│   │   ├── 04-ingress.yaml               # Traefik Ingress & Webhook route
│   │   └── 05-application.yaml           # Multi-cluster GitOps Application
│   │
│   └── sample-app/                       # Sample App Service & Deployment (UMail Admin)
│       ├── 01-namespace.yaml             # Namespace 'admin'
│       ├── 02-deployment.yaml            # 2 Replicas, rollingUpdate, probes, secrets
│       ├── 03-service.yaml               # ClusterIP Service (port 80 -> 3000)
│       └── 04-ingressroute.yaml          # Traefik IngressRoute with TLS
│
└── examples/                             # 💡 Pipeline Templates
    ├── .woodpecker-golang.yaml           # Go build & test pipeline
    └── .woodpecker-nodejs.yaml           # Node.js build & test pipeline
```

---

## 📑 Documentation Index

1. **[Quick Setup](docs/00-quick-setup.md)** — Deploy everything in 3 minutes.
2. **[Architecture & Concepts](docs/01-architecture-and-concepts.md)** — How Woodpecker and ArgoCD collaborate.
3. **[GitHub OAuth Setup](docs/02-github-oauth-setup.md)** — Connect your GitHub account to Woodpecker.
4. **[HashiCorp Vault Integration](docs/03-vault-integration.md)** — Secure secret storage and injection.
5. **[Installation Guide](docs/04-installation-guide.md)** — Complete step-by-step installation instructions.
6. **[Pipeline Syntax Guide](docs/05-pipeline-syntax-guide.md)** — Write `.woodpecker.yaml` pipelines with Kaniko.
7. **[User & Admin Management](docs/06-user-and-admin-management.md)** — Add team members and grant admin rights.

---

## 🔒 Security Summary

* **Rootless Builds**: Pipelines use Google's Kaniko. No `/var/run/docker.sock` mount required.
* **Least-Privilege**: Woodpecker CI cannot deploy to the cluster; it only updates Git. Only ArgoCD deploys to Kubernetes.
* **Team Whitelist**: Only members of `unity-workspace-org` can log in (`WOODPECKER_ORGS`).
* **Sub-Second GitOps**: GitHub webhook syncs ArgoCD in `<1ms`.

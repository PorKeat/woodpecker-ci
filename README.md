# Woodpecker CI + ArgoCD GitOps Documentation

A production-grade, lightweight, and secure CI/CD platform running natively on Kubernetes (`Node1-3`), integrated with HashiCorp Vault (`10.1.18.8`) and exposed at **[https://woodpecker.unity-workspace.com/](https://woodpecker.unity-workspace.com/)**.

---

## 📁 Repository Structure

```text
woodpecker-ci/
├── docs/                                 # 📖 Documentation Guides
│   ├── 01-architecture-and-concepts.md   # Architectural overview & components
│   ├── 02-github-oauth-setup.md          # GitHub OAuth application setup
│   ├── 03-vault-integration.md           # HashiCorp Vault secrets injection
│   ├── 04-installation-guide.md          # Step-by-step Helm installation guide
│   ├── 05-pipeline-syntax-guide.md       # .woodpecker.yaml syntax & Kaniko builds
│   └── 06-user-and-admin-management.md   # WOODPECKER_ADMIN, ORGS & team permissions
│
├── manifests/                            # 📦 Production Kubernetes Manifests
│   ├── woodpecker/
│   │   ├── values.yaml                   # Pinned Woodpecker Helm values
│   │   └── ingress.yaml                  # Traefik Ingress (root / and /woodpecker)
│   └── argocd/
│       ├── values.yaml                   # Pinned ArgoCD Helm values
│       ├── ingress.yaml                  # Traefik Ingress & Webhook route
│       └── application.yaml              # Multi-cluster GitOps Application spec
│
└── examples/                             # 💡 Example Pipelines
    ├── .woodpecker-golang.yaml           # Go test & build pipeline
    └── .woodpecker-nodejs.yaml           # Node.js test & build pipeline
```

---

## 📑 Table of Contents

1. [Architecture & Concepts](docs/01-architecture-and-concepts.md) — How Woodpecker CI and ArgoCD work together, execution models, and comparisons to Jenkins.
2. [GitHub OAuth Setup](docs/02-github-oauth-setup.md) — 30-second guide to configuring the GitHub OAuth App.
3. [HashiCorp Vault Integration](docs/03-vault-integration.md) — Storing RPC secrets, GitHub OAuth tokens, and injecting them into Kubernetes.
4. [Installation & Deployment Guide](docs/04-installation-guide.md) — Complete copy-paste commands to install ArgoCD and Woodpecker CI.
5. [Pipeline Syntax Guide](docs/05-pipeline-syntax-guide.md) — Reference for `.woodpecker.yaml` syntax, multi-stage pipelines, and Kaniko container builds.
6. [User, Team & Admin Management](docs/06-user-and-admin-management.md) — `WOODPECKER_ADMIN`, `WOODPECKER_ORGS`, and promoting teammates.
7. [Kubernetes Manifests](manifests/) — Production YAML templates for Helm and Traefik.
8. [Pipeline Examples](examples/) — Example `.woodpecker.yaml` files for Node.js and Go.

---

## 🔒 Security Highlights

- **Rootless Container Builds:** Pipelines use Google's **Kaniko** executor. Pipelines never mount `/var/run/docker.sock` or require privileged Docker-in-Docker daemons.
- **Least-Privilege CI:** Woodpecker CI never receives Kubernetes cluster admin credentials. It only commits the new image tag to Git. Only ArgoCD deploys to the cluster.
- **Centralized Vault Storage:** All admin passwords, pre-computed bcrypt hashes, and RPC encryption secrets are persisted in HashiCorp Vault on `10.1.18.8`.
- **Instant Webhooks:** Sub-second sync (< 1ms) between GitHub push events and ArgoCD GitOps reconciliations.

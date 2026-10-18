# Woodpecker CI + ArgoCD GitOps Documentation

A production-grade, lightweight, and secure CI/CD platform running natively on Kubernetes (`Node1-3`), integrated with HashiCorp Vault (`10.1.18.8`) and protected behind Traefik VPN-only Ingress (`https://10.1.16.200/argocd` and `https://10.1.16.200/woodpecker`).

---

## 📑 Table of Contents

1. [Architecture & Concepts](01-architecture-and-concepts.md) — How Woodpecker CI and ArgoCD work together, execution models, and comparisons to Jenkins & Tekton.
2. [GitHub OAuth Setup](02-github-oauth-setup.md) — 1-minute step-by-step guide to registering your GitHub OAuth App with exact callback URLs.
3. [HashiCorp Vault Integration](03-vault-integration.md) — How credentials, RPC tokens, and secrets are stored in Vault and injected into Kubernetes.
4. [Installation & Deployment Guide](04-installation-guide.md) — Complete copy-paste commands to install ArgoCD and Woodpecker CI via Helm with pinned secure images.
5. [Pipeline Syntax Guide](05-pipeline-syntax-guide.md) — Detailed reference for `.woodpecker.yaml` syntax, steps, multi-stage pipelines, Kaniko container builds, and GitOps triggers.
6. [Ready-to-Use Examples](examples/) — Pre-configured YAML templates for Node.js, Go, Python, Traefik Ingress, and ArgoCD manifests.

---

## 🏗️ High-Level Architecture

```
   ┌──────────────────────────────────────────────┐
   │            GitHub (Source Code)              │
   │  • Developer Pushes Code                     │
   │  • .woodpecker.yaml triggers build           │
   └──────────────────────┬───────────────────────┘
                          │ Webhook / API Trigger
                          ▼
   ┌──────────────────────────────────────────────┐
   │         Woodpecker CI (on Kubernetes)        │
   │  • Server (v3.19.0) coordinates pipeline     │
   │  • Agent (v3.19.0) spawns isolated K8s pods  │
   │  • Kaniko builds container image rootlessly   │
   │  • Pushes image to Container Registry (GHCR) │
   └──────────────────────┬───────────────────────┘
                          │ Bumps Image Tag in Git
                          ▼
   ┌──────────────────────────────────────────────┐
   │          GitOps Repository (GitHub)          │
   │  • apps/my-app/deployment.yaml               │
   │  • Records exact commit hash audit trail     │
   └──────────────────────┬───────────────────────┘
                          │ Real-time Git Sync
                          ▼
   ┌──────────────────────────────────────────────┐
   │             ArgoCD (GitOps Engine)           │
   │  • Reconciles Git state with K8s cluster     │
   │  • Automated zero-downtime rolling update    │
   │  • Instant rollback on Git revert            │
   └──────────────────────────────────────────────┘
```

---

## 🔒 Security Highlights

- **VPN-Only Access:** Both ArgoCD and Woodpecker CI dashboards are restricted to private subnets (`10.0.0.0/8`, `100.64.0.0/10`, `172.16.0.0/12`, `192.168.0.0/16`) via Traefik `ipAllowList` middleware.
- **Rootless Container Builds:** Pipelines use Google's **Kaniko** executor. Pipelines never mount `/var/run/docker.sock` or require privileged Docker-in-Docker (DinD) daemons.
- **Least-Privilege CI:** Woodpecker CI **never** receives Kubernetes cluster admin credentials. It only commits the new image tag to Git. Only ArgoCD touches the cluster.
- **Centralized Vault Storage:** All admin passwords, pre-computed bcrypt hashes, and RPC encryption secrets are persisted in HashiCorp Vault on `10.1.18.8`.
- **Pinned Secure Versions:** All container images use audited, immutable tags (ArgoCD `v3.5.5`, Woodpecker `v3.19.0`, Kaniko `v1.23.2-debug`).

---

## 🌐 Connected Kubernetes Clusters (Multi-Cluster GitOps)

ArgoCD is configured as a centralized management plane controlling multiple downstream Kubernetes clusters:

| Cluster Name | API Server URL | Description / Workloads |
| :--- | :--- | :--- |
| `in-cluster` | `https://kubernetes.default.svc` | Local cluster (`Node1-3`, Traefik VIP `10.1.16.200`) hosting ArgoCD & Woodpecker CI |
| `mail-cluster` | `https://10.1.15.11:6443` | 3-node cluster (`k8s-1` to `k8s-3` on `10.1.15.11-13`) for Mail services |
| `api-gateway-cluster` | `https://10.1.18.4:6443` | 3-node cluster (`api-gateway1` to `3` on `10.1.18.4,12,13`) for API Gateway services |
| `jitsi-meet-cluster` | `https://10.1.18.10:6443` | 3-node cluster (`jitsi-meet1` to `3` on `10.1.18.9-11`) for Video Conferencing |

### Deploying an Application to a Specific Cluster:
In your ArgoCD `Application` manifest, target any cluster by its friendly name:
```yaml
spec:
  destination:
    name: mail-cluster        # Or: api-gateway-cluster, jitsi-meet-cluster, in-cluster
    namespace: default
```

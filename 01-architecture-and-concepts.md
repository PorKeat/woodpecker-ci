# 01 — Architecture & Core Concepts

## 1. What is Woodpecker CI?

**Woodpecker CI** is a modern, lightweight, community-driven Continuous Integration engine derived from Drone CI. It is written in pure Go, has zero Java runtime requirements, and is specifically architected for containerized environments and Kubernetes.

### Woodpecker CI vs. Jenkins vs. Tekton

| Metric / Feature | Woodpecker CI | Jenkins | Tekton Pipelines |
| :--- | :--- | :--- | :--- |
| **Pipeline Language** | Standard YAML (`.woodpecker.yaml`) | Groovy DSL (`Jenkinsfile`) | Verbose K8s CRDs (`TaskRun`, `PipelineRun`) |
| **Runtime Footprint** | **~40 MB RAM** (Go binary) | **2.5 GB – 4 GB RAM** (Java JVM) | **~300 MB RAM** (Multiple CRD controllers) |
| **Execution Model** | Ephemeral Kubernetes Pods per step | JVM Agents or dynamic pod plugins | Pods created via CRDs per task |
| **Maintenance** | Single Helm release, zero plugin rot | Hundreds of brittle third-party plugins | Heavy maintenance of CRDs & tasks |
| **Startup Speed** | Milliseconds | 2–5 minutes | Fast |
| **Security Surface** | Minimal attack surface, rootless | High (historical CVEs, plugin vulnerabilities) | Minimal attack surface |

---

## 2. Core Components of Woodpecker CI

### 1. Woodpecker Server (`woodpecker-server`)
- Coordinates the CI queue, manages pipeline lifecycles, and stores build logs.
- Serves the Web Dashboard and REST API.
- Handles OAuth authentication and webhooks from GitHub.
- Communicates securely with Agents using gRPC encrypted via `WOODPECKER_SERVER_RPC_SECRET`.

### 2. Woodpecker Agent (`woodpecker-agent`)
- Polls the Server for pending pipeline jobs over gRPC.
- Uses the **Kubernetes Backend** (`WOODPECKER_BACKEND=kubernetes`).
- For each step defined in `.woodpecker.yaml`, the agent dynamically contacts the Kubernetes API server and provisions an isolated pod with the required container image.
- Once the step finishes (success or failure), the agent captures logs, transmits them to the server, and automatically terminates and cleans up the pod.

---

## 3. The GitOps Architecture: Woodpecker CI + ArgoCD

In traditional (legacy) CI/CD setups, the CI tool (like Jenkins or GitLab runner) was given full `cluster-admin` `kubeconfig` credentials to run `kubectl apply` directly against production clusters.

### The Security Flaw with Legacy CI:
If a malicious pull request or compromised npm/pip dependency executes code inside the CI pipeline, attackers could steal cluster-admin tokens and compromise the entire Kubernetes cluster.

### The Modern GitOps Solution:
```
[ Developer ]
      │ Push Code
      ▼
[ App Repository ] ────────────► [ Woodpecker CI Pipeline ]
                                          │
                                          ├─ 1. Run Unit/Integration Tests
                                          ├─ 2. Build Image via Kaniko (Rootless)
                                          ├─ 3. Push Image: ghcr.io/...:sha-abc1234
                                          │
                                          ▼ 4. Bump Image Tag in GitOps Repo
                                 [ GitOps Infrastructure Repo ]
                                          │
                                          │ (Watches Git Repository)
                                          ▼
                                 [ ArgoCD in Cluster ]
                                          │
                                          ▼ Reconcile & Rollout
                                 [ Kubernetes Production Pods ]
```

### Why This Architecture Wins:
1. **Zero Cluster Permissions in CI:** Woodpecker CI never holds a `kubeconfig` or cluster token. It only needs write access to the GitOps repo.
2. **Immutable Audit Log:** Every production change is a Git commit signed with author and timestamp.
3. **One-Click Rollback:** If a release has a bug, simply revert the Git commit in the GitOps repository. ArgoCD immediately detects the revert and restores the previous stable image in seconds.

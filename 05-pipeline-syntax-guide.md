# 05 — Woodpecker Pipeline Syntax Guide

Woodpecker pipelines are declared in a `.woodpecker.yaml` (or `.woodpecker/` directory) file at the root of your application repository. The syntax is clean, intuitive, and similar to GitHub Actions and Drone CI.

---

## 1. Pipeline Structure Anatomy

A Woodpecker pipeline consists of:
1. **`when` (Triggers / Conditions):** When the pipeline runs (branch, event, tag, files modified).
2. **`steps` (Execution Tasks):** Sequential or parallel container steps.
3. **`services` (Sidecar Containers):** Ephemeral databases or caches (e.g. Postgres, Redis) accessible during test runs.

```yaml
when:
  event: [push, pull_request]
  branch: [main, develop]

steps:
  - name: lint
    image: golangci/golangci-lint:v1.59-alpine
    commands:
      - golangci-lint run

  - name: test
    image: golang:1.22-alpine
    commands:
      - go test -v -cover ./...
```

---

## 2. Advanced Features

### 2.1 Using Secrets in Pipelines
Secrets configured in Woodpecker's UI or CLI can be injected as environment variables:

```yaml
steps:
  - name: deploy-notify
    image: curlimages/curl:latest
    commands:
      - curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"Build complete!\"}" $$SLACK_WEBHOOK_URL
    environment:
      SLACK_WEBHOOK_URL:
        from_secret: slack_webhook
```

### 2.2 Ephemeral Database Services
If your integration tests require a real PostgreSQL database, Woodpecker runs it as a sidecar container sharing the localhost network:

```yaml
services:
  - name: postgres
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: test_user
      POSTGRES_PASSWORD: test_password
      POSTGRES_DB: test_db

steps:
  - name: integration-tests
    image: node:20-alpine
    commands:
      - npm test
    environment:
      DATABASE_URL: postgres://test_user:test_password@postgres:5432/test_db
```

### 2.3 Daemonless Container Builds with Kaniko
In Kubernetes, mounting `/var/run/docker.sock` is a major security vulnerability because any container with Docker socket access can escape to root on the host node.

Google's **Kaniko** builds Dockerfiles inside an unprivileged container without requiring a Docker daemon:

```yaml
steps:
  - name: build-and-push
    image: gcr.io/kaniko-project/executor:v1.23.2-debug
    commands:
      - /kaniko/executor \
          --context=dir://. \
          --dockerfile=Dockerfile \
          --destination=ghcr.io/porkeat/my-service:${CI_COMMIT_SHA:0:7}
    environment:
      DOCKER_CONFIG:
        from_secret: docker_config
```

---

## 3. Built-In Environment Variables Reference

Woodpecker automatically exposes these variables inside every step:

| Variable | Description | Example |
| :--- | :--- | :--- |
| `CI_COMMIT_SHA` | Full 40-character commit hash | `a1b2c3d4e5f6...` |
| `${CI_COMMIT_SHA:0:7}` | Short 7-character commit hash | `a1b2c3d` |
| `CI_COMMIT_BRANCH` | Target branch | `main` |
| `CI_COMMIT_TAG` | Git tag (if triggered by a tag) | `v1.2.0` |
| `CI_COMMIT_AUTHOR` | Git author username | `PorKeat` |
| `CI_COMMIT_MESSAGE` | Commit subject line | `fix: resolve auth timeout` |
| `CI_PIPELINE_NUMBER` | Pipeline sequence number | `42` |
| `CI_PIPELINE_EVENT` | Trigger event | `push`, `pull_request`, `tag` |
| `CI_REPO` | Full repository name | `PorKeat/backend-api` |

---

## 4. Multi-Stage Pipeline Example (Test -> Build -> GitOps Deploy)

```yaml
when:
  event: [push]
  branch: main

steps:
  # Stage 1: Quality Checks
  - name: unit-tests
    image: node:20-alpine
    commands:
      - npm ci
      - npm run test:coverage

  # Stage 2: Security Linting
  - name: security-audit
    image: aquasec/trivy:latest
    commands:
      - trivy fs --exit-code 0 --severity HIGH,CRITICAL .

  # Stage 3: Build & Publish Image
  - name: publish-image
    image: gcr.io/kaniko-project/executor:v1.23.2-debug
    commands:
      - /kaniko/executor \
          --context=dir://. \
          --dockerfile=Dockerfile \
          --destination=ghcr.io/porkeat/my-service:${CI_COMMIT_SHA:0:7} \
          --destination=ghcr.io/porkeat/my-service:latest
    environment:
      DOCKER_CONFIG:
        from_secret: ghcr_auth

  # Stage 4: Trigger ArgoCD via GitOps Repo Bump
  - name: sync-argocd
    image: alpine/git:v2.45.2
    commands:
      - git config --global user.name "Woodpecker CI"
      - git config --global user.email "ci@local"
      - git clone https://$$GITHUB_TOKEN@github.com/PorKeat/gitops-deployments.git
      - cd gitops-deployments
      - sed -i "s|image: ghcr.io/porkeat/my-service:.*|image: ghcr.io/porkeat/my-service:${CI_COMMIT_SHA:0:7}|g" apps/my-service/deployment.yaml
      - git commit -am "chore(release): bump my-service to ${CI_COMMIT_SHA:0:7} [skip ci]"
      - git push origin main
    environment:
      GITHUB_TOKEN:
        from_secret: github_deploy_token
```

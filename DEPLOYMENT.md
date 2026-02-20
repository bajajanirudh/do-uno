# Deployment Guide — Task Management API

> Enterprise-grade deployment documentation for the Task Management API.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Local Docker Build & Run](#local-docker-build--run)
- [Push to DockerHub](#push-to-dockerhub)
- [Deploy to Kubernetes (Raw Manifests)](#deploy-to-kubernetes-raw-manifests)
- [Deploy to Kubernetes (Helm)](#deploy-to-kubernetes-helm)
- [Horizontal Pod Autoscaling](#horizontal-pod-autoscaling)
- [Ingress & TLS Setup](#ingress--tls-setup)
- [CI/CD Workflow Explanation](#cicd-workflow-explanation)
- [GitHub Environments](#github-environments)
- [Image Scanning Security](#image-scanning-security)
- [Required GitHub Secrets](#required-github-secrets)
- [Security Best Practices Implemented](#security-best-practices-implemented)
- [Production Observability](#production-observability)
- [Production Scalability](#production-scalability)
- [Why This Setup Is Enterprise-Ready](#why-this-setup-is-enterprise-ready)
- [Assumptions & Prerequisites](#assumptions--prerequisites)

---

## Prerequisites

| Tool       | Minimum Version | Purpose                     |
|------------|-----------------|-----------------------------|
| Docker     | 20.10+          | Container builds            |
| kubectl    | 1.26+           | Kubernetes CLI              |
| Kubernetes | 1.26+           | Orchestration platform      |
| Helm       | 3.12+           | Package management for K8s  |
| Git        | 2.30+           | Version control             |
| Python     | 3.11            | Local development only      |

Cluster-level requirements:

- **Metrics Server** — required for HPA CPU-based autoscaling.
- **NGINX Ingress Controller** — required for Ingress resources.
- **cert-manager** (optional) — for automated TLS certificate provisioning.
- **Prometheus Operator** (optional) — for ServiceMonitor-based scraping.

---

## Local Docker Build & Run

### Build

```bash
docker build -t task-api:latest .
```

### Run

```bash
docker run -d \
  --name task-api \
  -p 8000:8000 \
  --read-only \
  --tmpfs /tmp \
  task-api:latest
```

The `--read-only` flag mirrors the production `readOnlyRootFilesystem` SecurityContext. The `--tmpfs /tmp` provides a writable scratch area.

### Verify

```bash
curl http://localhost:8000/health
```

### Cleanup

```bash
docker stop task-api && docker rm task-api
```

---

## Push to DockerHub

```bash
docker login

docker tag task-api:latest <your-dockerhub-username>/task-api:latest
docker tag task-api:latest <your-dockerhub-username>/task-api:$(git rev-parse HEAD)

docker push <your-dockerhub-username>/task-api:latest
docker push <your-dockerhub-username>/task-api:$(git rev-parse HEAD)
```

Always push a SHA-tagged image alongside `latest` so every deployment is traceable to a specific commit.

---

## Deploy to Kubernetes (Raw Manifests)

Raw YAML manifests live in `k8s/` and can be applied directly for environments that do not use Helm.

### 1. Update the image reference

Edit `k8s/deployment.yaml` and replace `your-dockerhub-username/task-api:latest` with your actual image path.

### 2. Apply manifests

```bash
kubectl apply -f k8s/
```

### 3. Verify

```bash
kubectl rollout status deployment/task-api
kubectl get pods -l app=task-api
kubectl get svc task-api
kubectl get hpa task-api
```

### 4. Port-forward for local access

```bash
kubectl port-forward svc/task-api 8000:80
curl http://localhost:8000/health
```

---

## Deploy to Kubernetes (Helm)

The Helm chart in `helm/task-api/` is the recommended deployment method. It parameterizes every resource and supports per-environment overrides.

### Install

```bash
helm install task-api ./helm/task-api
```

### Install with overrides

```bash
helm install task-api ./helm/task-api \
  --set image.repository=myregistry/task-api \
  --set image.tag=abc123def \
  --set ingress.host=api.mycompany.com \
  --set ingress.tls.secretName=mycompany-tls
```

### Upgrade

```bash
helm upgrade task-api ./helm/task-api \
  --set image.tag=$(git rev-parse HEAD) \
  --wait --timeout 120s
```

### Uninstall

```bash
helm uninstall task-api
```

### Render templates locally (dry-run)

```bash
helm template task-api ./helm/task-api
```

### Key values you can override

| Value                                    | Default                                | Description                  |
|------------------------------------------|----------------------------------------|------------------------------|
| `image.repository`                       | `your-dockerhub-username/task-api`     | Image registry path          |
| `image.tag`                              | `latest`                               | Image tag                    |
| `replicaCount`                           | `3`                                    | Baseline replicas            |
| `autoscaling.enabled`                    | `true`                                 | Enable HPA                   |
| `autoscaling.maxReplicas`                | `10`                                   | HPA ceiling                  |
| `autoscaling.targetCPUUtilizationPercentage` | `70`                               | Scale-up CPU threshold       |
| `ingress.enabled`                        | `true`                                 | Enable Ingress resource      |
| `ingress.host`                           | `task-api.local`                       | Ingress hostname             |
| `ingress.tls.secretName`                 | `task-api-tls`                         | TLS certificate secret       |
| `resources.requests.cpu`                 | `100m`                                 | CPU request                  |
| `resources.limits.memory`                | `512Mi`                                | Memory ceiling               |

---

## Horizontal Pod Autoscaling

The HPA is defined in `k8s/hpa.yaml` (raw) and `helm/task-api/templates/hpa.yaml` (Helm).

- **API version:** `autoscaling/v2`
- **Min replicas:** 3 — guarantees high-availability baseline.
- **Max replicas:** 10 — caps resource consumption during traffic spikes.
- **Target metric:** 70% average CPU utilization across all pods.

### How it works

The Metrics Server collects CPU utilization from each pod's `cgroup`. When average utilization exceeds 70%, the HPA controller increases the replica count (up to 10). When utilization drops, it gradually scales back down to 3.

### Verify HPA status

```bash
kubectl get hpa task-api
kubectl describe hpa task-api
```

### Prerequisite

The Kubernetes Metrics Server must be running:

```bash
kubectl top pods
```

If this command fails, deploy the Metrics Server first.

---

## Ingress & TLS Setup

### Ingress

The Ingress resource routes external HTTPS traffic to the ClusterIP service. It uses the `nginx` IngressClass and includes:

- **SSL redirect** — all HTTP requests are redirected to HTTPS.
- **Rewrite target** — strips path prefixes if sub-path routing is added later.
- **Backend protocol** — explicitly set to HTTP since TLS terminates at the Ingress controller.

### TLS certificate

The Ingress references a Kubernetes TLS secret named `task-api-tls`. You can create it manually:

```bash
kubectl create secret tls task-api-tls \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

Or automate provisioning with cert-manager by adding an annotation:

```yaml
cert-manager.io/cluster-issuer: letsencrypt-prod
```

### DNS

Point `task-api.local` (or your real domain) to the Ingress controller's external IP:

```bash
kubectl get svc -n ingress-nginx
```

---

## CI/CD Workflow Explanation

The pipeline is defined in `.github/workflows/ci-cd.yml` and consists of three sequential jobs:

```
push to main ──▶ Test ──▶ Build, Scan & Push ──▶ Deploy (Helm)
pull request  ──▶ Test (only)
```

### Job 1 — Test

- Runs on every push and pull request to `main`.
- Sets up Python 3.11 with pip caching.
- Installs dependencies and runs `pytest`.
- Blocks all downstream jobs if any test fails.

### Job 2 — Build, Scan & Push

- Runs only on pushes to `main`.
- Builds the Docker image using Buildx with GitHub Actions layer cache (`type=gha`).
- **Scans the image with Trivy** before pushing. The pipeline fails if any HIGH or CRITICAL CVE is detected.
- Authenticates to DockerHub and pushes two tags: `latest` and the full commit SHA.

### Job 3 — Deploy

- Runs only after a successful build on `main`.
- Targets the `production` GitHub Environment (requires approval if configured).
- Uses concurrency control (`production-deploy` group) to prevent overlapping deployments.
- Installs Helm and runs `helm upgrade --install` with the commit SHA as the image tag.
- Waits for the rollout to succeed (120 s timeout).

---

## GitHub Environments

The deploy job targets the **production** GitHub Environment. This enables:

- **Approval gates** — require manual approval before deploying to production.
- **Environment secrets** — scope secrets (e.g. `KUBE_CONFIG`) to the production environment only.
- **Deployment history** — GitHub tracks every deployment with commit, status, and timestamp.

### Setup

1. Go to **Settings → Environments → New environment**.
2. Name it `production`.
3. (Optional) Add required reviewers.
4. (Optional) Restrict to the `main` branch.

### Concurrency control

```yaml
concurrency:
  group: production-deploy
  cancel-in-progress: true
```

If a new commit is pushed while a deployment is in progress, the older deployment is cancelled. This prevents stale deployments from completing after a newer version is already queued.

---

## Image Scanning Security

Every Docker image is scanned with [Trivy](https://trivy.dev/) before being pushed to the registry.

- **Severity threshold:** HIGH and CRITICAL.
- **Unfixed vulnerabilities:** Ignored (`ignore-unfixed: true`) so the pipeline does not block on CVEs without available patches.
- **Pipeline behavior:** The build job fails with exit code 1 if any qualifying vulnerability is found, preventing the image from reaching production.

This provides a shift-left security gate — vulnerabilities are caught before they ever enter the container registry or the cluster.

---

## Required GitHub Secrets

| Secret               | Scope          | Description                                              |
|----------------------|----------------|----------------------------------------------------------|
| `DOCKERHUB_USERNAME` | Repository     | Your DockerHub username                                  |
| `DOCKERHUB_TOKEN`    | Repository     | A DockerHub access token (not your account password)     |
| `KUBE_CONFIG`        | Environment    | Base64-encoded kubeconfig for the target cluster         |

### Creating the KUBE_CONFIG secret

```bash
cat ~/.kube/config | base64 | pbcopy   # macOS
cat ~/.kube/config | base64 -w 0       # Linux (copy output)
```

Paste into **Settings → Environments → production → Environment secrets**.

---

## Security Best Practices Implemented

### Non-root container

The Dockerfile creates a dedicated `appuser` (UID 1000) and switches to it before the CMD. The pod-level `runAsNonRoot: true` enforces this at the Kubernetes level — the kubelet will refuse to start the container if it attempts to run as root.

### Read-only root filesystem

`readOnlyRootFilesystem: true` prevents any writes to the container filesystem. A writable `/tmp` is provided via an `emptyDir` volume. This mitigates entire classes of attacks that rely on writing malicious binaries to the container.

### Dropped capabilities

All Linux capabilities are dropped (`drop: ALL`). The container cannot use `chown`, `setuid`, `net_raw`, or any other privileged syscall. `allowPrivilegeEscalation: false` prevents gaining capabilities after process start.

### Minimal base image

`python:3.11-slim` is used instead of the full Debian image, reducing attack surface and image size by ~800 MB.

### Multi-stage build

Dependencies are compiled in a `builder` stage. Only the final packages are copied into the production image. Build tools, caches, and intermediate layers are excluded.

### Image vulnerability scanning

Trivy scans every image in CI before it is pushed to the registry. HIGH and CRITICAL CVEs block the pipeline.

### Resource limits

Kubernetes resource requests and limits prevent any single pod from consuming unbounded CPU or memory:

- **Requests:** 100 m CPU / 128 Mi memory
- **Limits:** 500 m CPU / 512 Mi memory

### Image tagging strategy

Every image is tagged with the full Git commit SHA. The deploy job always uses the SHA tag, never `latest`. This ensures:

- Deployments are deterministic and auditable.
- Rollbacks target a specific commit.
- Image pull skew across replicas is impossible.

### TLS termination at Ingress

All external traffic is encrypted via TLS at the NGINX Ingress controller. The `ssl-redirect` annotation ensures HTTP requests are upgraded to HTTPS.

---

## Production Observability

### ServiceMonitor (Prometheus)

A `ServiceMonitor` resource (`k8s/servicemonitor.yaml`) is included for clusters running the Prometheus Operator. It scrapes the `/health` endpoint every 30 seconds and exposes:

- Application health status
- Uptime
- Total task count
- Memory usage estimate

### Kubernetes-native observability

The three-probe configuration provides granular observability:

| Probe     | Purpose                                                        |
|-----------|----------------------------------------------------------------|
| Startup   | Prevents liveness checks from killing pods that are still initializing. Allows up to 300 s (30 failures × 10 s). |
| Liveness  | Restarts unresponsive pods automatically.                      |
| Readiness | Removes unhealthy pods from the Service endpoint list so they receive no traffic. |

---

## Production Scalability

### Horizontal scaling

The HPA automatically adjusts replica count between 3 and 10 based on CPU utilization. Combined with the rolling update strategy (`maxSurge: 1`, `maxUnavailable: 1`), scaling events complete quickly without service disruption.

### Resource efficiency

Resource requests inform the scheduler's bin-packing decisions, ensuring pods land on nodes with sufficient capacity. Limits protect against noisy-neighbor effects in multi-tenant clusters.

### Concurrency-safe deployments

The CI/CD concurrency group ensures only one deployment runs at a time. If a fast-follow commit is pushed, the previous deploy is cancelled rather than competing for the same resources.

### Helm-based lifecycle management

Helm tracks release revisions, enabling instant rollback:

```bash
helm rollback task-api 1
```

---

## Why This Setup Is Enterprise-Ready

| Capability                     | Implementation                                               |
|--------------------------------|--------------------------------------------------------------|
| **Container security**         | Non-root, read-only FS, all capabilities dropped, slim image |
| **Supply chain security**      | Trivy CVE scanning in CI, SHA-pinned image tags              |
| **High availability**          | 3 replica minimum, rolling updates, startup/liveness/readiness probes |
| **Auto-scaling**               | HPA scales to 10 replicas on CPU pressure                    |
| **TLS encryption**             | NGINX Ingress with TLS termination and forced redirect       |
| **Observability**              | ServiceMonitor for Prometheus, three-probe health model      |
| **GitOps-compatible**          | Helm chart with value overrides, deterministic SHA deploys   |
| **Deployment safety**          | GitHub Environment approvals, concurrency control            |
| **Infrastructure-as-Code**     | All resources defined in version-controlled YAML/Helm        |
| **Rollback capability**        | Helm revision history, SHA-tagged images                     |

---

## Assumptions & Prerequisites

1. **In-memory storage:** The application stores data in memory. Each pod maintains its own state. For true production use, we will need to add a shared datastore (e.g., PostgreSQL, Redis).
2. **Kubernetes cluster:** A running cluster (v1.26+) with `kubectl` and Helm access is required.
3. **Metrics Server:** Must be deployed for HPA to function.
4. **NGINX Ingress Controller:** Must be deployed for Ingress resources to be reconciled.
5. **Prometheus Operator:** Optional. Required only if using the ServiceMonitor for metrics scraping.
6. **DockerHub registry:** The pipeline pushes to DockerHub. Swap the login action and image references for ECR, GCR, GHCR, or any OCI-compliant registry.
7. **Single namespace:** Manifests do not specify a namespace and deploy to the current context's default.
8. **cert-manager:** Optional. For automated TLS certificate lifecycle, deploy cert-manager and add the appropriate annotation to the Ingress.

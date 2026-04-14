# AI Puzzle Solver — GitOps Configuration Repository

> **This repository contains zero application source code.**
> It is the single source of truth for all Kubernetes/OpenShift infrastructure
> declarations that deploy the AI Puzzle Solver backend to production.

---

## Why This Repository Exists

Modern cloud-native deployments separate two distinct concerns:

| Concern | Repository |
|---|---|
| **Application source code** | `ai-Puzzle-Solver-backend` |
| **Infrastructure declarations** | **This repo** (`ai-Puzzle-Solver-gitops`) |

This separation — known as **GitOps** — means the cluster state is always
derived from Git, never from manual `kubectl apply` commands. Every change
is reviewed, versioned, audited, and reproducible.

**Benefits:**
- **Auditability** — every deployment change has a commit, an author, and a timestamp
- **Rollback in seconds** — revert a bad deploy with `git revert`
- **Drift detection** — ArgoCD continuously reconciles; any manual cluster change is auto-corrected
- **Zero-touch deployments** — merge to `main` → cluster updates itself

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Developer Workflow                        │
│                                                                  │
│  1. Push code to ai-Puzzle-Solver-backend                       │
│  2. CI builds & pushes new image to Docker Hub                  │
│  3. Update image tag in this repo (overlays/prod)               │
│  4. ArgoCD detects the change and syncs the cluster             │
└─────────────────────────────────────────────────────────────────┘

         GitHub                    OpenShift Cluster
   ┌──────────────┐           ┌─────────────────────────┐
   │  This Repo   │◄──watch───│        ArgoCD            │
   │  (GitOps)    │           │  (openshift-gitops ns)   │
   └──────────────┘           └────────────┬────────────┘
                                           │ applies
                               ┌───────────▼────────────┐
                               │  ai-puzzle-solver-prod  │
                               │  ┌───────────────────┐  │
                               │  │  Deployment (x2)  │  │
                               │  │  Service          │  │
                               │  │  Route (TLS)      │  │
                               │  └───────────────────┘  │
                               └────────────────────────┘
```

---

## Repository Structure

```
ai-Puzzle-Solver-gitops/
├── argocd/
│   └── application.yaml          # ArgoCD Application CRD
├── base/                         # Shared base manifests (env-agnostic)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── route.yaml
│   ├── secret.yaml               # Template only — never commit real secrets
│   └── kustomization.yaml
└── overlays/
    └── prod/                     # Production-specific overrides
        ├── deployment-patch.yaml # Replicas: 2, higher resource limits
        └── kustomization.yaml    # Namespace: ai-puzzle-solver-prod
```

### Key Design Decisions

**Kustomize over Helm** — The application has a single deployment target with
straightforward configuration. Kustomize patches are simpler to audit and
require no templating language.

**OpenShift Route over Ingress** — Red Hat OpenShift uses its own `Route`
resource for HTTP/HTTPS traffic routing with native TLS edge termination.
Standard Kubernetes `Ingress` objects are not used.

**Non-root container (UID 1001)** — OpenShift's default restricted Security
Context Constraints (SCC) reject containers that run as root. The backend
image is built with `USER 1001` and all writable paths redirected to `/tmp`
to comply with arbitrary UID assignment.

**emptyDir volumes for model cache** — The `BAAI/bge-m3` (~1.2 GB) and
DeepFace models are downloaded at first startup into ephemeral volumes.
This keeps the container image lean while ensuring OpenShift's read-only
root filesystem policy is respected.

---

## What Was Done

### Phase 1 — Backend Containerization (`ai-Puzzle-Solver-backend`)

- Vendored **DeepFace v0.0.99** face recognition library directly into the
  backend source tree, eliminating a runtime PyPI dependency on an unstable
  package
- Wrote an OpenShift-compliant multi-stage `Dockerfile` targeting `python:3.11-slim`
- Layered pip installs (PyTorch → sentence-transformers → TensorFlow → CV →
  DeepFace → app deps) so Docker cache reuse is maximized on rebuilds
- Added `pymilvus[milvus_lite]` for local SQLite-backed vector storage
- Set `--start-period 300s` on the health check to accommodate first-run
  model downloads from HuggingFace
- Built and pushed a **multi-platform manifest** (`linux/amd64` + `linux/arm64`)
  to Docker Hub:
  ```
  harudavv/mazeai-showcases:ai-puzzle-solver-backend
  ```

### Phase 2 — GitOps Repository (This Repo)

- Initialized Kustomize base with `Deployment`, `Service`, `Route`, and
  `Secret` manifests
- Created `overlays/prod` with production-scale overrides (2 replicas,
  higher CPU/memory limits)
- Authored the **ArgoCD Application CRD** with automated sync and self-healing
  pointed at this repository

---

## What Is Next

### Immediate — Before First Deploy

- [ ] **Fill in secrets** — Do NOT commit real API keys to this repo.
  Apply the secret manually to the cluster:
  ```bash
  oc create secret generic ai-puzzle-solver-secrets \
    --from-literal=ANTHROPIC_API_KEY=<your-key> \
    --from-literal=OPENAI_API_KEY=<your-key> \
    --from-literal=PUZZLEMIND_LLM_PROVIDER=claude \
    -n ai-puzzle-solver-prod
  ```
  Then delete `base/secret.yaml` or replace it with an
  [External Secrets Operator](https://external-secrets.io/) reference.

- [ ] **Create the namespace on the cluster**
  ```bash
  oc new-project ai-puzzle-solver-prod
  ```

- [ ] **Install ArgoCD** (if not already present on the cluster)
  ```bash
  oc apply -k https://github.com/argoproj/argo-cd/manifests/cluster-install
  ```
  On OpenShift, the Red Hat GitOps Operator is preferred:
  ```bash
  # Via OperatorHub in the OpenShift web console
  # Operator: "Red Hat OpenShift GitOps"
  ```

- [ ] **Apply the ArgoCD Application**
  ```bash
  oc apply -f argocd/application.yaml
  ```
  ArgoCD will immediately sync `overlays/prod` to the cluster.

### Short-term Improvements

- [ ] **Persistent model cache** — Replace `emptyDir` volumes with a
  `PersistentVolumeClaim` so `BAAI/bge-m3` and DeepFace models survive
  pod restarts and the 5-minute cold start is eliminated
- [ ] **Image tag pinning** — Replace `ai-puzzle-solver-backend` (mutable tag)
  with an immutable digest in `overlays/prod/deployment-patch.yaml`:
  ```yaml
  image: harudavv/mazeai-showcases@sha256:<digest>
  ```
- [ ] **CI/CD integration** — Add a GitHub Actions workflow to the backend
  repo that builds, pushes, and automatically opens a PR against this repo
  updating the image digest
- [ ] **Horizontal Pod Autoscaler** — Add an `HPA` manifest targeting 70% CPU
  utilization to scale replicas dynamically under load

### Long-term

- [ ] **Secrets management** — Integrate HashiCorp Vault or AWS Secrets Manager
  via the External Secrets Operator
- [ ] **Multi-environment overlays** — Add `overlays/staging` for pre-production
  validation before changes reach `overlays/prod`
- [ ] **Network Policy** — Restrict pod-to-pod traffic with `NetworkPolicy`
  manifests once the full service mesh topology is defined
- [ ] **Observability** — Add `ServiceMonitor` CRD for Prometheus scraping of
  the `/metrics` endpoint

---

## Quick Reference

### Verify the Kustomize output locally (no cluster needed)
```bash
kubectl kustomize overlays/prod
```

### Trigger a manual ArgoCD sync
```bash
argocd app sync ai-puzzle-solver-backend
```

### Check application health
```bash
argocd app get ai-puzzle-solver-backend
```

### Roll back to a previous revision
```bash
argocd app history ai-puzzle-solver-backend
argocd app rollback ai-puzzle-solver-backend <revision-id>
```

---

## Container Image

| Property | Value |
|---|---|
| Registry | Docker Hub |
| Image | `harudavv/mazeai-showcases:ai-puzzle-solver-backend` |
| Platforms | `linux/amd64`, `linux/arm64` |
| Base | `python:3.11-slim` |
| Run as | UID 1001 (non-root, OpenShift restricted SCC compliant) |
| Exposed port | `8008` |

---

## Related Repositories

| Repository | Purpose |
|---|---|
| [`Escanor4323/ai-Puzzle-Solver-gitops`](https://github.com/Escanor4323/ai-Puzzle-Solver-gitops) | This repo — infrastructure declarations |
| `ai-Puzzle-Solver-backend` | FastAPI backend, face recognition, LLM integration |
| `ai-Puzzle-Solver-frontend` | Frontend application |

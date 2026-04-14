# AI Puzzle Solver — GitOps Configuration Repository

> **This repository contains zero application source code.**
> It is the single source of truth for all Kubernetes/OpenShift infrastructure
> declarations that deploy the AI Puzzle Solver backend to production.

---

## Why This Repository Exists

Modern cloud-native deployments separate two distinct concerns:

| Concern | Repository |
|---|---|
| **Application source code** | [`ai-Puzzle-Solver-backend`](https://github.com/Escanor4323/ai-Puzzle-Solver-backend) |
| **Infrastructure declarations** | **This repo** (`ai-Puzzle-Solver-gitops`) |

This separation — known as **GitOps** — means the cluster state is always
derived from Git, never from manual `kubectl apply` commands. Every change
is reviewed, versioned, audited, and reproducible.

**Benefits:**
- **Auditability** — every deployment change has a commit, an author, and a timestamp
- **Rollback in seconds** — revert a bad deploy with `git revert`
- **Drift detection** — ArgoCD continuously reconciles; any manual cluster change is auto-corrected
- **Zero-touch deployments** — merge to `main` in the backend → image builds → cluster updates itself

---

## Full System Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Developer Workflow                             │
│                                                                       │
│  1. Push / merge to main in ai-Puzzle-Solver-backend                 │
│  2. GitHub Actions builds linux/amd64 + linux/arm64 image            │
│  3. Image pushed to Docker Hub  harudavv/mazeai-showcases:<sha>      │
│  4. CI job clones this repo, updates base/deployment.yaml image tag  │
│  5. CI commits "chore: bump backend image tag to <sha>"              │
│  6. ArgoCD detects the new commit and syncs the cluster              │
└──────────────────────────────────────────────────────────────────────┘

  ai-Puzzle-Solver-backend          ai-Puzzle-Solver-gitops
  ┌───────────────────────┐         ┌──────────────────────┐
  │  FastAPI + DeepFace   │─CI────► │  Kustomize manifests │
  │  .github/workflows/   │  push   │  overlays/prod/      │
  │    ci.yaml            │  tag    │  argocd/             │
  └───────────────────────┘         └──────────┬───────────┘
                                               │ watched by
                                    ┌──────────▼───────────┐
         Docker Hub                 │       ArgoCD          │
  ┌───────────────────────┐         │  (openshift-gitops)  │
  │ harudavv/mazeai-      │◄────────│  auto-sync + heal    │
  │ showcases:<sha>       │  pull   └──────────┬───────────┘
  └───────────────────────┘                    │ applies
                                    ┌──────────▼───────────┐
                                    │  ai-puzzle-solver-   │
                                    │  prod (namespace)    │
                                    │  ┌────────────────┐  │
                                    │  │ Deployment x2  │  │
                                    │  │ Service        │  │
                                    │  │ Route (TLS)    │  │
                                    │  └────────────────┘  │
                                    └──────────────────────┘
```

---

## Repository Structure

```
ai-Puzzle-Solver-gitops/
├── argocd/
│   └── application.yaml              # ArgoCD Application CRD
├── base/                             # Shared base manifests (env-agnostic)
│   ├── deployment.yaml               # CI auto-updates image tag here
│   ├── service.yaml
│   ├── route.yaml                    # OpenShift Route (TLS edge)
│   ├── secret.yaml                   # Template only — never commit real secrets
│   └── kustomization.yaml
└── overlays/
    └── prod/                         # Production-specific overrides
        ├── kustomization.yaml        # Namespace + images[] tag override
        └── deployment-patch.yaml     # Replicas: 2, higher resource limits
```

---

## CI/CD Pipeline (Prompt 3)

The pipeline lives in `ai-Puzzle-Solver-backend/.github/workflows/ci.yaml`.

### Trigger
Runs on every push to `main`. Pull requests run only the build step (no push, no GitOps update).

### Jobs

```
push to main
     │
     ▼
┌─────────────────────┐
│  build-and-push     │  Builds linux/amd64 + linux/arm64 image
│                     │  Tags: ai-puzzle-solver-backend-<7-char-sha>
│                     │  Pushes to Docker Hub
└──────────┬──────────┘
           │ on success
           ▼
┌─────────────────────┐
│  update-gitops      │  Clones this repo using GITOPS_PAT
│                     │  Updates image tag via yq
│                     │  Commits "chore: bump backend image tag to <sha>"
│                     │  Pushes back to this repo → ArgoCD syncs
└─────────────────────┘
```

### Required GitHub Secrets (set in `ai-Puzzle-Solver-backend` repo settings)

| Secret | Where to get it |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username (`harudavv`) |
| `DOCKERHUB_TOKEN` | Docker Hub → Account Settings → Security → Access Tokens |
| `GITOPS_PAT` | GitHub → Settings → Developer Settings → Personal Access Tokens (Fine-grained) — needs `Contents: Read & Write` on this repo only |

---

## What Was Done

### Phase 1 — Backend Containerization

- Vendored **DeepFace v0.0.99** face recognition library into `vendor/` — no runtime PyPI dependency
- OpenShift-compliant `Dockerfile`: `USER 1001`, non-root, all writable paths redirect to `/tmp`
- Added `pymilvus[milvus_lite]` for local SQLite-backed vector storage
- Health check `--start-period 300s` to accommodate first-run model downloads (~1.2 GB BGE-M3)
- Multi-platform manifest (`linux/amd64` + `linux/arm64`) on Docker Hub:
  `harudavv/mazeai-showcases:ai-puzzle-solver-backend`

### Phase 2 — GitOps Repository

- Kustomize `base/` with `Deployment`, `Service`, `Route`, `Secret`
- `overlays/prod` with 2-replica patch and `images[]` override block for CI tag injection
- ArgoCD Application CRD with automated sync and self-healing

### Phase 3 — CI/CD Bridge

- GitHub Actions workflow in `ai-Puzzle-Solver-backend/.github/workflows/ci.yaml`
- Job 1: builds and pushes multi-platform image tagged with Git SHA
- Job 2: clones this repo, patches `base/deployment.yaml` with `yq`, commits and pushes
- GitHub Actions layer cache (`cache-from/to: type=gha`) for fast rebuilds

---

## What Is Next

### Immediate — Before First Deploy

- [ ] **Add secrets to backend repo** (`DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `GITOPS_PAT`)
  in GitHub → `ai-Puzzle-Solver-backend` → Settings → Secrets and variables → Actions

- [ ] **Apply cluster secrets** — never commit real keys to this repo:
  ```bash
  oc create secret generic ai-puzzle-solver-secrets \
    --from-literal=ANTHROPIC_API_KEY=<your-key> \
    --from-literal=OPENAI_API_KEY=<your-key> \
    --from-literal=PUZZLEMIND_LLM_PROVIDER=claude \
    -n ai-puzzle-solver-prod
  ```

- [ ] **Create the namespace**
  ```bash
  oc new-project ai-puzzle-solver-prod
  ```

- [ ] **Install Red Hat OpenShift GitOps Operator** via OperatorHub in the OpenShift web console

- [ ] **Apply the ArgoCD Application**
  ```bash
  oc apply -f argocd/application.yaml
  ```

### Short-term Improvements

- [ ] **Persistent model cache** — replace `emptyDir` with a `PersistentVolumeClaim` to
  eliminate the 5-minute cold start caused by BGE-M3 and DeepFace model downloads
- [ ] **Image digest pinning** — CI currently writes a mutable SHA tag; migrate to
  immutable digest (`@sha256:...`) for full reproducibility
- [ ] **Frontend CI pipeline** — mirror this pipeline for `ai-Puzzle-Solver-frontend`
- [ ] **PR previews** — add a staging ArgoCD Application pointing to `overlays/staging`
  that deploys on feature branches

### Long-term

- [ ] **External Secrets Operator** — replace `secret.yaml` template with an
  `ExternalSecret` CRD pulling from HashiCorp Vault or AWS Secrets Manager
- [ ] **Network Policy** — restrict pod-to-pod traffic once service mesh topology is final
- [ ] **Horizontal Pod Autoscaler** — scale on CPU/memory automatically
- [ ] **Observability** — add `ServiceMonitor` for Prometheus scraping of `/metrics`

---

## Quick Reference

```bash
# Dry-run the prod overlay locally (no cluster needed)
kubectl kustomize overlays/prod

# Manually trigger ArgoCD sync
argocd app sync ai-puzzle-solver-backend

# Check application health and sync status
argocd app get ai-puzzle-solver-backend

# Roll back to a previous deployment
argocd app history ai-puzzle-solver-backend
argocd app rollback ai-puzzle-solver-backend <revision-id>
```

---

## Container Image

| Property | Value |
|---|---|
| Registry | Docker Hub |
| Image | `harudavv/mazeai-showcases:ai-puzzle-solver-backend-<sha>` |
| Platforms | `linux/amd64`, `linux/arm64` |
| Base | `python:3.11-slim` |
| Run as | UID 1001 (OpenShift restricted SCC compliant) |
| Exposed port | `8008` |

---

## Related Repositories

| Repository | Purpose |
|---|---|
| [`Escanor4323/ai-Puzzle-Solver-gitops`](https://github.com/Escanor4323/ai-Puzzle-Solver-gitops) | This repo — infrastructure declarations |
| [`Escanor4323/ai-Puzzle-Solver-backend`](https://github.com/Escanor4323/ai-Puzzle-Solver-backend) | FastAPI backend, face recognition, LLM integration |
| [`Escanor4323/ai-Puzzle-Solver-frontend`](https://github.com/Escanor4323/ai-Puzzle-Solver-frontend) | Frontend application |

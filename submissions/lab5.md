# Lab 5 — CI/CD & GitOps

## Overview
In this lab I implemented a full CI/CD pipeline using **GitHub Actions** and set up **GitOps** with **ArgoCD** for the QuickTicket application.

**Goal achieved:**  
- Automated container image builds and pushes to GitHub Container Registry (ghcr.io)  
- ArgoCD installation and configuration  
- GitOps deployment from Git repository to Kubernetes cluster  
- Tested self-healing via Git (rollback)

---

## Task 1 — CI Pipeline + ArgoCD Setup

### 1. CI Workflow (`.github/workflows/ci.yml`)

Created a GitHub Actions workflow that:
- Triggers on push to `main`
- Builds Docker images for `gateway`, `events`, and `payments`
- Pushes them to `ghcr.io` using commit SHA as tag

**Workflow successfully executed** with green status.

### 2. Images in GitHub Container Registry
Images were successfully built and pushed:
- `quickticket-gateway`
- `quickticket-events`
- `quickticket-payments`

### 3. Updated Kubernetes Manifests
Updated `k8s/*.yaml` files to use images from `ghcr.io`:
- Changed `imagePullPolicy` to `Always`
- Added `imagePullSecrets` for private registry access

### 4. ArgoCD Installation
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
ArgoCD UI was accessed via port-forward (https://localhost:8443).
### 5. ArgoCD Application Creation
Created quickticket Application pointing to:

Repository: https://github.com/foidkiller/SRE-Intro.git
Path: k8s
Destination: default namespace
Sync Policy: Automated + Prune + Self Heal

### 6. GitOps Loop Verification
Made changes in Git => ArgoCD automatically synced them to the cluster.

## Task 2 — Rollback via GitOps
Deploy Broken Version
Modified k8s/gateway.yaml to use a non-existent image tag:
YAMLimage: ghcr.io/foidkiller/quickticket-gateway:does-not-exist
Pushed the change. ArgoCD synced it => Gateway pod went into ImagePullBackOff / ErrImagePull state.
Rollback
```Bash
git revert HEAD --no-edit
git push origin main
```
ArgoCD automatically detected the revert and restored the previous working version.
Recovery time: ~1–2 minutes after git push.

## Bonus Task — Automated Image Tag Update
Understood the concept and partially implemented the logic in CI workflow (using sed to update image tags in manifests + commit back to repository with CI skip condition).

## Lab 5 Summary
In this laboratory I:

Wrote a complete GitHub Actions CI pipeline for building and pushing Docker images
Configured ArgoCD for declarative GitOps deployments
Experienced the full GitOps loop: code change so CI build so ArgoCD sync so deployment
Successfully tested rollback by reverting a bad deployment via Git (without using kubectl)
Gained practical experience with modern SRE/DevOps practices

## Conclusion:
GitOps with ArgoCD provides excellent visibility, reliability, and rollback capabilities compared to manual deployments. This is one of the most valuable labs in the course.

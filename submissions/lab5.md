# Lab 5 — CI/CD & GitOps


## Objective
The goal of this laboratory work was to implement a complete CI/CD pipeline using GitHub Actions and establish a GitOps workflow with ArgoCD for the QuickTicket application.

## Task 1 — CI Pipeline and ArgoCD Setup (6 points)

### 1.1 GitHub Actions CI Pipeline
I created the file `.github/workflows/ci.yml` from scratch. The workflow is triggered on every push to the `main` branch. It builds and pushes Docker images for all three services (`gateway`, `events`, and `payments`) to GitHub Container Registry (`ghcr.io`).


### 1.2 Verification of Pushed Images
After the pipeline completed successfully, I verified that all three container images were published:

```bash
gh api user/packages?package_type=container --jq '.[].name'
```
Images successfully pushed:

quickticket-gateway
quickticket-events
quickticket-payments

1.3 Kubernetes Manifests Update
I updated all manifests in the k8s/ directory to use images from the GitHub Container Registry:

Changed imagePullPolicy from Never to Always
Added imagePullSecrets to each Deployment

Example (excerpt from gateway.yaml):
```YAML
spec:
  containers:
    - name: gateway
      image: ghcr.io/foidkiller/quickticket-gateway:${{ github.sha }}
      imagePullPolicy: Always
  imagePullSecrets:
    - name: ghcr-secret
```
1.4 ArgoCD Installation
```Bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Available deployment/argocd-server -n argocd --timeout=300s
```
1.5 ArgoCD Application
I created the ArgoCD Application using the CLI:
```Bash
argocd app create quickticket \
  --repo https://github.com/foidkiller/SRE-Intro.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --upsert
```
Verification:
```Bash
argocd app get quickticket
```
Result:

Sync Status: Synced
Health Status: Healthy

1.6 GitOps Loop Test
I made a visible change in k8s/gateway.yaml (added a version label), committed and pushed it. ArgoCD automatically detected the change and synchronized it to the cluster.
Verification command:
```Bash
kubectl get deployment gateway -o jsonpath='{.metadata.labels.version}'
```
# Output: v2

Task 2 — Rollback via GitOps (4 points)
2.1 Deploying a Broken Version
I intentionally changed the image tag in k8s/gateway.yaml to a non-existent version:
```YAML
image: ghcr.io/foidkiller/quickticket-gateway:does-not-exist
```
After pushing the changes, ArgoCD attempted to sync, and the gateway pod entered ImagePullBackOff state.
Proof:
```Bash
argocd app get quickticket
kubectl get pods -l app=gateway
```
2.2 Rollback
I performed a rollback using Git:
```Bash
git revert HEAD --no-edit
git push origin main
```
ArgoCD automatically synchronized the reverted state, and the application returned to a healthy condition.
Recovery time: Approximately 1.5 minutes after the push.
Git history:
```Bash
git log --oneline -3
```

Bonus Task — Automated Image Tag Update (2 points)
I implemented logic in the CI workflow that automatically updates image tags in the Kubernetes manifests after building new images and commits the changes back to the repository (with protection against infinite CI loops).

Conclusion
In this lab I successfully implemented a modern CI/CD and GitOps workflow:

Automated container image building and publishing using GitHub Actions
Declarative deployment and synchronization using ArgoCD
Full GitOps cycle: code change so build so deploy
Safe rollback capability through Git

This laboratory demonstrated the power and reliability of GitOps practices, which provide better traceability, reproducibility, and recovery compared to manual deployments.

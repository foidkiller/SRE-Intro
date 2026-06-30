# Lab 7 — Progressive Delivery: Canary Deployments


## Introduction
In this lab I installed Argo Rollouts, converted the gateway Deployment into a Canary Rollout, tested manual promotion and abort, and also implemented a multi-step canary strategy with automated analysis (Bonus).

## Task 1 — Manual Canary Deployment

### Argo Rollouts Installation
Successfully installed Argo Rollouts controller in the `argo-rollouts` namespace.

### Converting gateway to Rollout
Changed `k8s/gateway.yaml`:
- `kind: Deployment` → `kind: Rollout`
- Added `apiVersion: argoproj.io/v1alpha1`
- Added canary strategy with steps (20% → pause → 60% → pause → 100%)

**Replicas:** 5 (needed for meaningful traffic splitting)

### Canary Deployment Process
- Updated image/environment to simulate a new version (`APP_VERSION=v2-canary`)
- Rollout started and paused at **20%** (1 canary pod out of 5)
- Observed 4 stable pods + 1 canary pod

### Manual Promotion
Used `kubectl patch` to continue the rollout through all steps until 100%.

### Bad Version + Abort
- Deployed a bad version (`APP_VERSION=v3-BAD-VERSION`)
- Rollout paused at 20% with the bad canary pod
- Executed **abort** — the bad pod was immediately terminated
- All traffic instantly returned to stable pods

**Comparison with Lab 5:** Abort with Argo Rollouts is much faster and safer than `git revert` + ArgoCD sync.

## Task 2 — Multi-step Canary

Updated the strategy to a more gradual rollout:

```yaml
strategy:
  canary:
    steps:
      - setWeight: 20
      - pause: {duration: 60s}
      - setWeight: 40
      - pause: {duration: 60s}
      - setWeight: 60
      - pause: {duration: 60s}
      - setWeight: 80
      - pause: {duration: 30s}
      - setWeight: 100
```
Observed the rollout progressing step by step using kubectl get rollout gateway and pod list.
Bonus Task — Automated Canary Analysis

Created AnalysisTemplate (gateway-error-rate) that queries Prometheus for error rate
Added analysis step into the Rollout strategy
Tested both:
Good version → auto-promote
Bad version → auto-abort based on error rate


This allows fully automated progressive delivery with safety checks.
Conclusion
This lab gave me hands-on experience with progressive delivery. I really liked how Argo Rollouts allows safe deployment of new versions with the ability to quickly abort if something goes wrong.
Canary deployments combined with automated analysis are a powerful tool for reducing risk in production.

# Lab 10 — SRE Portfolio & Reliability Review

## Load Test Results

Load testing was performed using Locust running inside the Kubernetes cluster. The goal was to identify the application's capacity limits and observe how system reliability changed under increasing load.

| Users | Ramp Rate | Duration |   RPS |    p50 |    p95 |    p99 | 5xx Error Rate | Observation                               |
| ----: | --------: | -------: | ----: | -----: | -----: | -----: | -------------: | ----------------------------------------- |
|    10 | 2 users/s |     60 s |  7.70 |   8 ms |  16 ms |  37 ms |             0% | Stable performance with no errors         |
|    50 | 5 users/s |     60 s | 31.24 | 170 ms | 740 ms | 990 ms |         15.34% | Increased latency and elevated error rate |

### Breaking Point

The system began to degrade at approximately **50 concurrent users** (about **31 requests per second**).

The following symptoms were observed:

* Increased response latency.
* Growing number of HTTP 500, 502, and 503 responses.
* Higher request failure rate.
* Reduced overall system responsiveness.

These observations indicate that the application reached its current capacity limit under sustained load.

---

# DORA Metrics

### Deployment Frequency

Deployment frequency was **medium**. Throughout the course, application updates were deployed regularly using the GitOps workflow.

### Lead Time for Changes

Lead time was typically only a few minutes. After pushing changes to Git, ArgoCD automatically synchronized and deployed the updated manifests.

### Change Failure Rate

The estimated change failure rate was **10–15%**, primarily due to failed canary deployments and intentionally injected failures during chaos engineering experiments.

### Time to Restore Service

The average time to restore service ranged from **10 to 30 seconds**. Automatic rollback through Argo Rollouts significantly reduced recovery time after unsuccessful deployments.

---

# Top Three Reliability Risks

## 1. Downstream Service Dependencies

Failures in Redis or the Payments service directly affect request processing in the Gateway. Introducing circuit breakers and graceful degradation mechanisms would improve resilience.

## 2. Database Storage Reliability

Without a PersistentVolumeClaim, PostgreSQL data can be lost when the database pod is recreated. Persistent storage is essential for production deployments.

## 3. Monitoring Coverage

Current monitoring focuses primarily on application errors. Additional alerts should cover latency, dependency health, and partial service degradation.

---

# Toil Identification

| Manual Task                   | Frequency             | Proposed Automation             |
| ----------------------------- | --------------------- | ------------------------------- |
| Port forwarding to Prometheus | Frequent              | Shell script or Makefile target |
| Redis database cleanup        | Before each load test | Automated cleanup job           |
| Manual chaos experiments      | Several times         | Chaos Mesh or LitmusChaos       |

These repetitive operational tasks consume engineering time without providing long-term value and should be automated whenever possible.

---

# Monitoring Gaps

Several important monitoring capabilities are still missing:

* Alert for high p99 latency on the `/pay` endpoint.
* Dedicated health alert for the Payments service.
* Redis availability alert.
* Metrics for circuit breaker activations and downstream failures.
* Service Level Objective (SLO) monitoring for latency and availability.

Adding these alerts would improve incident detection and reduce mean time to identify problems.

---

# Capacity Plan for Double Traffic

If application traffic doubled, the following scaling strategy would be appropriate:

| Component  | Recommended Scaling                               |
| ---------- | ------------------------------------------------- |
| Gateway    | 8–10 replicas                                     |
| Events     | 4–5 replicas                                      |
| Payments   | 4 replicas with circuit breaker support           |
| Redis      | Replication enabled                               |
| PostgreSQL | Persistent storage with regular automated backups |

Estimated infrastructure cost for a small Kubernetes cluster would be approximately **$30–50 per month**, depending on the cloud provider and storage configuration.

---

# Highest-Priority Improvement

The highest-priority improvement would be implementing **circuit breakers together with latency-based SLO alerts**.

Circuit breakers would prevent cascading failures when downstream services become unavailable, while latency alerts would allow operators to detect performance degradation before it becomes visible to end users.

---

# Final Reflection

Throughout this course, I gained practical experience with modern SRE and cloud-native technologies, including:

* Docker and containerization
* Kubernetes orchestration
* Prometheus and Grafana monitoring
* GitOps using ArgoCD
* Progressive delivery with Argo Rollouts
* Chaos Engineering
* Database migration, backup, and disaster recovery
* Reliability engineering principles and operational best practices

This portfolio demonstrated how infrastructure automation, observability, deployment strategies, and resilience techniques work together to build reliable distributed systems. The course significantly improved my understanding of Site Reliability Engineering and the practical challenges involved in operating production services.

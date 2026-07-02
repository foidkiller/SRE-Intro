# Lab 8 — Chaos Engineering: Break Things on Purpose


## Introduction
In this lab I conducted three chaos experiments on QuickTicket. For each experiment I wrote a hypothesis first, injected a failure, observed the system using Prometheus and kubectl, and documented the results.

## Experiment 1 — Pod Kill Under Load

**Hypothesis (before running):**  
"If I kill one gateway pod under load, Kubernetes will quickly start a new one. Traffic will be redistributed to the remaining pods, and there should be only a short spike in errors."

**Execution:**
```bash
VICTIM=$(kubectl get pods -l app=gateway -o name | head -1)
echo "Killing $VICTIM at $(date)"
kubectl delete $VICTIM
```
Observations:
```Bash
kubectl get pods -l app=gateway -w
```
 New pod appeared in ~8 seconds. Full recovery (5/5 Ready) in 22 seconds.
Error rate during incident:
```Bash
kubectl exec -n monitoring deployment/prometheus -- wget -qO- \
  'http://localhost:9090/api/v1/query?query=sum(increase(gateway_requests_total{status=~"5.."}[5m]))'
```
 Brief spike in 5xx errors (about 12 failed requests during the gap).
Traffic distribution:
4 stable pods continued serving traffic while the new pod was starting.
Conclusion: Hypothesis was mostly correct. Self-healing worked well, but there was a short period of degraded availability.
Improvement: Add pod disruption budget or increase replica count.

Experiment 2 — Payment Latency Injection
Hypothesis (before running):
"If payments service starts responding with 2000ms latency, only the /pay endpoint will slow down. Read operations (/events, /reserve) will remain fast because they don’t call payments."
Execution:
```Bash
kubectl set env deployment/payments PAYMENT_LATENCY_MS=2000
kubectl rollout status deployment/payments --timeout=30s
```
Observations:

/events: ~180ms (normal)
/reserve: ~250ms (normal)
/pay: ~2.1 seconds (clear slowdown)

Prometheus p99 latency:
```Bash
kubectl exec -n monitoring deployment/prometheus -- wget -qO- \
  'http://localhost:9090/api/v1/query?query=histogram_quantile(0.99, sum by (le, path) (rate(gateway_request_duration_seconds_bucket[5m])))'
```
 p99 for /pay jumped to ~2100ms, while other paths stayed under 400ms.
Error rate: Remained low (no timeout reached).
Conclusion: Hypothesis confirmed. Partial degradation was isolated to payment flow.
Improvement: Add latency-based SLO alert for /pay endpoint.

Experiment 3 — Redis Failure
Hypothesis (before running):
"If Redis is killed, read operations (/events) will continue working, but reservation (/reserve) and payment will fail because they depend on Redis for ticket holding."
Execution:
```Bash
kubectl scale deployment/redis --replicas=0
```
Observations:
```Bash
curl -s http://localhost:3080/health | python3 -m json.tool
```
redis: down, events: ok, payments: ok

/events so worked normally (200 OK)
/reserve so failed (503 or timeout)
/pay so also failed

After restore:
```Bash
kubectl scale deployment/redis --replicas=1
```
System recovered within 25 seconds.
Conclusion: Hypothesis was correct. The system showed graceful partial degradation.
Improvement: Implement circuit breaker in gateway for Redis-dependent calls.

Task 2 — Combined Failure Scenario
Scenario: Payments with 30% failure rate + 800ms latency + limited DB connections.
Execution:
```Bash
kubectl set env deployment/payments PAYMENT_FAILURE_RATE=0.3 PAYMENT_LATENCY_MS=800
kubectl set env deployment/events DB_MAX_CONNS=3
kubectl scale deployment/mixedload --replicas=3
```
Observations (after 3 minutes):

Overall error rate: ~28%
/pay: highest error rate and latency
/reserve: also degraded due to DB connection limit
Weakest link: Payments service + DB connection pool

Conclusion: Combined failures amplified each other. Payments was the main bottleneck.

Bonus Task — Resilience Improvement
Chosen weakness: Redis failure causing reservation failures.
Fix implemented: Added readinessProbe and livenessProbe to Redis + increased replicas: 2 for Redis.
Before fix: Many reservation failures during Redis downtime.
After fix: System tolerated short Redis unavailability much better.
Trade-off: Slightly higher resource usage.

Final Thoughts
Chaos Engineering helped me discover real weaknesses in the system (especially dependency on payments and Redis). I now better understand the importance of isolation, retries, and proper health checks.

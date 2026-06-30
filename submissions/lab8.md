# Lab 8 — Chaos Engineering: Break Things on Purpose


## Introduction
In this lab I practiced Chaos Engineering by designing and running three experiments on the QuickTicket system. I wrote hypotheses before each experiment, injected failures, observed the system behavior, and documented what I learned. I also ran a combined failure scenario.

## Experiment 1 — Pod Kill Under Load

**Hypothesis (written before running):**  
"If I delete one gateway pod while traffic is flowing, Kubernetes will quickly create a replacement pod, and users should see almost no impact — maybe just 1-2 failed requests at most — because the Service load balancer will distribute traffic to the remaining pods."

**Execution:**
```bash
VICTIM=$(kubectl get pods -l app=gateway -o name | head -1)
echo "Killing $VICTIM at $(date)"
kubectl delete $VICTIM
```
Observations:

A new pod started almost immediately.
It took about 18-22 seconds until we had 5/5 Running pods again.
During the gap, there were a few failed requests (error rate spiked briefly).
The remaining 4 pods absorbed most of the traffic.

Comparison with Hypothesis:
My hypothesis was mostly correct. The system recovered quickly thanks to Kubernetes self-healing, but there was a short period of elevated errors. I was surprised how fast the new pod became Ready.
Improvement idea: To improve resilience I would add a readiness probe with a short grace period or increase the number of replicas.

Experiment 2 — Payment Latency Injection
Hypothesis (written before running):
"If payments starts responding with 2000ms latency, only the /pay endpoint will become slow, while /events and /reserve will stay fast, because they don’t depend on payments. The gateway timeout should protect users from very long waits."
Execution:
```Bash
kubectl set env deployment/payments PAYMENT_LATENCY_MS=2000
kubectl rollout status deployment/payments --timeout=30s
```
Observations:

/events and /reserve remained fast (~0.1-0.3s).
/pay requests became slow (~2 seconds).
No significant increase in 5xx errors (because 2000ms < gateway timeout).
p99 latency for /pay path increased dramatically.

Comparison with Hypothesis:
Hypothesis was correct. The system degraded gracefully for payment-related operations while read paths stayed unaffected. This shows good isolation between services.
Improvement idea: Add a dedicated latency SLO alert for the /pay endpoint to catch slow payments before users notice.

Experiment 3 — Redis Failure
Hypothesis (written before running):
"If Redis goes down, users will still be able to list events (read-only), but reservation and payment will fail because they need Redis to hold tickets. The /health endpoint should report degraded state."
Execution:
```Bash
kubectl scale deployment/redis --replicas=0
```
Observations:

/events continued to work normally.
/reserve and /pay started failing.
/health showed events: ok, but payments and redis: down.
After restoring Redis (--replicas=1), the system recovered quickly.

Comparison with Hypothesis:
Completely correct. The system showed partial availability — read operations survived, write operations failed. This is actually good resilience design.
Improvement idea: Add a circuit breaker in the gateway for Redis-dependent operations to fail fast and return a nice error message.

Task 2 — Combined Failure Scenario
Scenario: Increased payment failure rate + latency + limited DB connections.
Execution:
```Bash
kubectl set env deployment/payments PAYMENT_FAILURE_RATE=0.3 PAYMENT_LATENCY_MS=800
kubectl set env deployment/events DB_MAX_CONNS=3
kubectl scale deployment/mixedload --replicas=3
```
Observations:

Overall error rate increased significantly.
/pay was the first and worst affected endpoint.
The system started returning many 5xx and slow responses.
Weakest link: The payments service + limited DB connection pool in events.

Conclusion: The payments service was the weakest link in this scenario. To make it more resilient I would add retries with backoff and better connection pool management.

Bonus Task — Resilience Improvement
I chose the Redis failure as the weakness to fix.
Change made: Increased Redis readiness probe timeout and added liveness probe.
Before fix: Many reservation failures when Redis was temporarily unavailable.
After fix: System tolerated short Redis hiccups much better.
Trade-off: Slightly longer startup time for events service.

Final Conclusion
Chaos Engineering showed me how the system behaves under real stress. I discovered that while Kubernetes provides good self-healing for pod failures, downstream service degradation (payments, Redis) still propagates to users. We need better isolation, circuit breakers, and more specific alerts.
This lab was very useful for understanding system resilience.

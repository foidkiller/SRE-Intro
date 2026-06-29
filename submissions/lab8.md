# Lab 8 - Chaos Engineering: Break Things on Purpose

## Experiment 1 - Pod Kill Under Load

### Hypothesis

If I delete one gateway pod during load, Kubernetes will create a new pod quickly and users will see very little impact.

### What I did

```bash
kubectl delete pod gateway-...
```

### Observations

* New pod was created in about 10 to 15 seconds.
* Number of pods returned to normal.
* There were very few failed requests.
* Kubernetes Service moved traffic to healthy pods.

### Result

My hypothesis was correct.

Kubernetes self-healing worked very well.

### Improvement

I would add a better readinessProbe so traffic goes only to fully ready pods.

---

## Experiment 2 - Payment Latency Injection

### Hypothesis

If payments service becomes slow (2000ms latency), only the `/pay` endpoint will be slow. Other endpoints should continue working normally.

### What I did

```bash
kubectl set env deployment/payments PAYMENT_LATENCY_MS=2000
```

### Observations

* `/events` was fast.
* `/reserve` was fast.
* `/pay` became much slower, about 2 seconds.
* Error rate did not increase much because timeout was not reached.

### Result

My hypothesis was correct.

The system was degraded only partially.

### Improvement

Add a separate SLO and alert for p95 and p99 latency on important endpoints like `/pay`.

---

## Experiment 3 - Redis Failure

### Hypothesis

If Redis is unavailable, users can still see events but cannot reserve tickets.

### What I did

```bash
kubectl scale deployment/redis --replicas=0
```

### Observations

* `/events` continued working.
* `/reserve` returned errors.
* `/health` showed degraded status.

### Result

My hypothesis was correct.

The system still worked partially.

### Improvement

Add circuit breaker and graceful degradation when Redis is unavailable.

---

## Task 2 - Combined Failure Scenario

### Scenario

* Payments latency = 2000ms
* Redis unavailable

### Observations

This was the worst scenario.

Users could not reserve tickets and payment requests were very slow.

### Conclusion

The system needs better resilience for downstream services.

Circuit breakers and better timeout settings would help.

---

## Bonus Task - Resilience Improvement

### Weakness

Slow payments service can affect user experience.

### Improvement

I would add retry logic and better timeout configuration.

### Before and After

After improvement, response time would be more stable and users would see fewer failures.

### Most Important Action

Add a circuit breaker in gateway for payments service.

This would stop problems from spreading to the whole application.

---

## Final Thoughts

In this lab I learned how to test failures on purpose and observe system behavior.

I tested:

* pod failure
* slow service response
* Redis failure
* multiple failures together

Chaos engineering helped me understand weak points in the system and how to improve reliability.

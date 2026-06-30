# Lab 11 — Advanced Microservice Patterns

## Introduction

The objective of this lab was to improve the resilience of the QuickTicket microservice application by implementing several common reliability patterns. During the lab, I added retries with exponential backoff, a notifications service, a circuit breaker, a rate limiter, and bulkhead isolation. Each feature was tested under failure conditions to evaluate its effectiveness.

---

# Task 1 — Notifications Service and Retries

## Notifications Service

A new **Notifications** service was created based on the existing Payments service. The service exposes a `/notify` endpoint and is invoked asynchronously after successful payment processing.

To avoid affecting user response time, notifications were implemented using a fire-and-forget approach. Notification failures do not block or fail the checkout request.

### Fire-and-Forget Test

Configuration:

* `NOTIFY_FAILURE_RATE=0.4`
* 30 checkout requests were executed.

### Observations

* All 30 checkout requests completed successfully.
* Notification failures did not affect the checkout workflow.
* Payment latency remained low because notification delivery was asynchronous.

### Conclusion

The fire-and-forget pattern successfully isolated notification failures from the critical payment path, improving overall application resilience.

---

## Retry Mechanism

A reusable `call_with_retry()` helper function was implemented using exponential backoff with randomized jitter.

### Test Configuration

* `PAYMENT_FAILURE_RATE=0.3`

### Observations

* Most failed payment requests succeeded after one or more retry attempts.
* The retry metric `gateway_retry_total{result="retried"}` increased as expected.
* Users experienced fewer failed payment requests despite transient downstream failures.

### Conclusion

The retry mechanism improved reliability during temporary service failures while reducing unnecessary request failures.

---

# Task 2 — Circuit Breaker and Rate Limiter

## Circuit Breaker

A simple circuit breaker state machine was implemented with three states:

* **CLOSED**
* **OPEN**
* **HALF_OPEN**

### Test Results

When the Payments service was configured to fail continuously:

* The circuit transitioned to the **OPEN** state.
* The Gateway immediately returned HTTP **503 Service Unavailable** responses instead of waiting for downstream timeouts.
* After the configured cooldown period (30 seconds), the circuit entered the **HALF_OPEN** state.
* Successful requests closed the circuit and normal processing resumed.

### Conclusion

The circuit breaker prevented repeated calls to an unhealthy dependency and significantly reduced unnecessary waiting time.

---

## Rate Limiter

A sliding-window rate limiter was implemented with a limit of **10 requests per second**.

### Test Results

A burst of 30 requests was generated.

Observed behavior:

* Requests above the configured limit returned **HTTP 429 Too Many Requests**.
* The response included the `Retry-After: 1` header.
* Requests within the configured rate limit continued to succeed normally.

### Conclusion

The rate limiter successfully protected the application from excessive request bursts while providing clients with guidance on when to retry.

---

# Bonus Task — Bulkhead Isolation

Bulkhead isolation was implemented by limiting the number of concurrent requests sent to the Payments service.

### Test Configuration

* `PAYMENT_LATENCY_MS=3000`

### Results

**Without Bulkhead**

* Slow payment requests occupied shared execution resources.
* The `/events` endpoint also experienced increased latency.
* Overall application responsiveness degraded.

**With Bulkhead**

* `/events` continued responding normally.
* `/pay` returned HTTP 503 when the concurrency limit was reached.
* Other application components remained responsive despite the slow dependency.

### Conclusion

Bulkhead isolation successfully contained the impact of a slow downstream service and prevented cascading performance degradation.

---

# Final Conclusion

This lab demonstrated several important resilience patterns commonly used in distributed systems:

* Retry with exponential backoff and jitter
* Fire-and-forget asynchronous notifications
* Circuit Breaker
* Sliding-window Rate Limiter
* Bulkhead Isolation

Together, these patterns significantly improved the application's ability to tolerate temporary failures, slow downstream services, and bursts of incoming traffic. They also reduced the likelihood of cascading failures and improved the overall reliability of the QuickTicket microservice architecture.

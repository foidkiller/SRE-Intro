# Lab 6 — Alerting & Incident Response



# Task 1 — Alerts, Runbook and Incident Response

## 1. Alert Rules in Grafana

### Alert 1 — QuickTicket High Error Rate (Critical)

| Parameter | Value |
|-----------|-------|
| **Name** | `QuickTicket High Error Rate` |
| **Severity** | Critical |
| **Condition** | IS ABOVE **10** |
| **Evaluate Every** | 1 minute |
| **Pending Period** | 2 minutes |
| **Label** | `severity=critical` |

### PromQL

```promql
sum(rate(gateway_requests_total{status=~"5.."}[5m]))
/
sum(rate(gateway_requests_total[5m]))
* 100 > 10
```

### Annotations

**Summary**

```text
High error rate detected: {{ $value | printf "%.2f" }}%
```

**Description**

```text
Gateway is returning too many 5xx errors.
Immediate investigation required.
```

---

### Alert 2 — QuickTicket SLO Burn Rate (Warning)

| Parameter | Value |
|-----------|-------|
| **Name** | `QuickTicket SLO Burn Rate` |
| **Severity** | Warning |
| **Condition** | IS ABOVE **6** |
| **Evaluate Every** | 1 minute |
| **Pending Period** | 5 minutes |
| **Label** | `severity=warning` |

### PromQL

```promql
(1 - (
  sum(rate(gateway_requests_total{status!~"5.."}[30m]))
  /
  sum(rate(gateway_requests_total[30m]))
))
/
(1 - 0.995)
> 6
```

---

# 2. Contact Point & Notification Policy

## Contact Point

**Type**

```
Webhook
```

**Endpoint**

```
https://webhook.site/b395b256-1ac3-4478-8193-ba329270ebf2
```

**Status**

✅ Successfully tested. Full JSON payload received.

---

## Notification Policy

| Setting | Value |
|---------|-------|
| Default Contact Point | `quickticket-alerts` |
| Group By | `alertname` |
| Group Wait | 30 seconds |
| Repeat Interval | 5 minutes |

---

# 3. Runbook — QuickTicket High Error Rate

## Alert

This alert fires when the **gateway 5xx error rate exceeds 10% for more than two minutes.**

---

## Diagnosis Steps

### 1. Check gateway health

```bash
curl -s http://localhost:3080/health | python3 -m json.tool
```

### 2. Check health of individual services

Verify that:

- Gateway
- Payments
- Events
- Reservations

are healthy.

### 3. Review recent logs

```bash
docker compose logs --tail=100 gateway payments
```

### 4. Verify Prometheus metrics

Check:

```text
payments_requests_total
```

---

## Common Causes & Mitigation

| Cause | Identification | Resolution |
|--------|---------------|------------|
| High `PAYMENT_FAILURE_RATE` | Payments service healthy but logs contain failures | Set environment variable to `0.0` and restart Payments |
| Payments container crashed | `docker compose ps` shows service stopped | `docker compose up -d payments` |
| Events service unavailable | Health endpoint fails | Restart Events service |
| Database issues | Connection errors in Events logs | Verify PostgreSQL and restart Events |

---

## Escalation

If the issue is **not resolved within 10 minutes**, notify the **team lead**.

---

# 4. Incident Simulation & Response

## Failure Injection

```bash
PAYMENT_FAILURE_RATE=0.8 \
docker compose \
-f docker-compose.yaml \
-f ../docker-compose.monitoring.yaml \
up -d payments
```

---

## Incident Timeline

| Time | Event |
|------|-------|
| **14:52:00** | Failure injected (`PAYMENT_FAILURE_RATE=0.8`) |
| **14:55:12** | Alert entered **Pending** |
| **14:57:45** | Alert changed to **Firing** |
| **14:57:50** | Webhook notification received |
| **14:58:10** | Runbook execution started |
| **14:59:20** | Root cause identified (Payments service) |
| **15:00:05** | Fix applied (`PAYMENT_FAILURE_RATE=0.0`) |
| **15:02:30** | Alert resolved |

---

## Result

From failure injection to alert firing took approximately **5 minutes 45 seconds**.

The delay is expected because Grafana evaluates alerts periodically and requires a pending period before firing. This configuration intentionally reduces false positives while slightly increasing detection time.

---

# 5. Proofs

The following requirements were successfully completed:

- ✅ Alert rules created in Grafana
- ✅ Alert rules tested successfully
- ✅ Webhook notification received
- ✅ JSON payload captured
- ✅ Runbook followed during a real incident

---

# Task 2 — Blameless Postmortem

## Incident

**High Error Rate Due to Payments Service Degradation**

| Field | Value |
|-------|-------|
| **Date** | June 17, 2026 |
| **Duration** | 10 minutes 30 seconds |
| **Severity** | SEV-3 |
| **Author** | Ravil Khusnutdinov |

---

## Summary

A configuration change increased the **Payments failure rate to 80%**, resulting in elevated **5xx Gateway errors**.

The issue was detected automatically by the SLO-based alerting system and resolved by following the established runbook.

---

## Timeline

The incident timeline is identical to the simulation described above.

---

## Root Cause

The monitoring system lacked a dedicated low-level alert for the Payments service.

As a result, the degradation propagated until it became visible through Gateway error rates.

Although the High Error Rate alert correctly detected the issue, the configured pending period delayed notification.

---

## What Went Well

- Alerting system detected the incident automatically.
- Webhook notification arrived immediately.
- Runbook provided clear diagnostic steps.
- Root cause was identified quickly.
- Recovery was completed within minutes.

---

## What Went Wrong

- No service-specific alert for Payments failures.
- Two-minute pending period delayed detection.
- Runbook lacked configuration and environment variable checks.

---

## Action Items

| Action | Owner | Priority | Status |
|---------|-------|----------|--------|
| Create dedicated alert for `PAYMENT_FAILURE_RATE` | Ravil | High | To Do |
| Reduce pending period to 90 seconds | Ravil | High | To Do |
| Extend runbook with configuration checks | Ravil | Medium | To Do |
| Add automated tests for failure injection | Team | Low | To Do |

---

## Most Important Action Item

**Create a dedicated Payments failure rate alert.**

### Reason

A service-specific alert would detect degradation before it impacts Gateway availability, SLO compliance, and user experience, significantly reducing mean time to detection (MTTD).

---

# Bonus Task — Cross-Tested Runbook

## Second Runbook

### Redis Unavailable (Reservations Failing)

### Alert

High Redis error rate or Redis connection failures.

---

## Diagnosis

### Verify Redis health

```bash
docker compose exec redis redis-cli ping
```

### Check application logs

Inspect Gateway and Events logs for Redis connection errors.

### Verify service status

Ensure Redis is running correctly in Docker or Kubernetes.

---

## Resolution

- Restart Redis container.
- Verify network connectivity.
- Scale Redis if experiencing high load.

---

## Cross-Test Result

A classmate successfully diagnosed and resolved the Redis outage using only this runbook.

**Time to recovery:** **4 minutes**

### Feedback Received

> Missing check for Redis memory usage.

### Improvement Added

The runbook was updated with an additional verification step:

```bash
docker stats
```

to monitor Redis memory consumption.

---

# Conclusion

This lab demonstrated a complete incident response workflow including:

- Grafana alert creation
- PromQL-based monitoring
- Webhook notifications
- Incident response using a structured runbook
- Blameless postmortem analysis
- Continuous improvement through actionable follow-up tasks

The monitoring configuration successfully detected service degradation, enabling rapid diagnosis and recovery while identifying opportunities to further reduce detection time in future incidents.

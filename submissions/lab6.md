# Lab 6 - Alerting & Incident Response

## Task 1 - Alerts, Runbook and Incident Response

### Alerts

I created these alerts in Grafana:

* **QuickTicket High Error Rate** (critical)
* **SLO Burn Rate** (warning)

For High Error Rate alert I used this PromQL:

```promql
sum(rate(gateway_requests_total{status=~"5.."}[5m])) / sum(rate(gateway_requests_total[5m])) * 100 > 10
```

Condition:

* Above 10
* For 2 minutes

### Contact Point

I used webhook.site as webhook contact point.

The alert test worked successfully.

### Runbook: High Error Rate

#### Alert

* Fires when Gateway 5xx error rate is more than 10% for 2 minutes
* Severity: Critical

#### Diagnosis Steps

1. Check health status:

```bash
curl -s http://localhost:3080/health | python3 -m json.tool
```

2. Check payments health:

```bash
curl -s http://localhost:8082/health
```

3. Check logs:

```bash
docker compose logs payments --tail=30
docker compose logs gateway --tail=30
```

4. Check environment variables in payments service.

#### Common Causes and Fixes

| Cause                  | How to identify          | Fix                                |
| ---------------------- | ------------------------ | ---------------------------------- |
| Payments service down  | payments health is down  | `docker compose start payments`    |
| High failure rate      | PAYMENT_FAILURE_RATE > 0 | Set value to 0 and restart service |
| Events service problem | events health is down    | Restart events                     |

#### Escalation

If problem is not fixed after 10 minutes, escalate to instructor or TA.

### Incident Timeline

* Failure injected with PAYMENT_FAILURE_RATE=0.5
* Alert fired after about 3 to 4 minutes
* Root cause found
* Fixed by restarting payments with normal settings

## Task 2 - Blameless Postmortem

### Postmortem: Payments Service Error Rate Spike

**Date:** June 17, 2026
**Duration:** About 12 minutes
**Severity:** SEV-2
**Author:** Student

### Summary

A configuration change increased payment failures. This caused more 5xx errors in gateway. Alert system detected the problem and response followed the runbook.

### Timeline

| Time     | Event                          |
| -------- | ------------------------------ |
| T+0      | Payment failure rate increased |
| T+3 min  | High Error Rate alert fired    |
| T+6 min  | Root cause found               |
| T+10 min | Service fixed                  |
| T+12 min | Alert resolved                 |

### Root Cause

There was not enough monitoring for payment service failure rate. Because of this, the problem affected gateway before it was detected.

### What Went Well

* Alert worked correctly
* Runbook was useful
* Problem was fixed quickly

### What Went Wrong

* Detection took some time
* No special alert for payments failures
* Load testing made troubleshooting harder

### Action Items

| Action                          | Owner    | Priority |
| ------------------------------- | -------- | -------- |
| Add alert for payments failures | SRE Team | High     |
| Reduce alert waiting time       | SRE Team | High     |
| Improve runbook                 | SRE Team | Medium   |
| Add failure scenario tests      | Dev Team | Medium   |

### Most Important Action

Add dedicated alert for payments service failures.

This will help find problems earlier and reduce impact on users.

## Bonus Task - Second Runbook

### Runbook: Redis Unavailable

#### Alert

Fires when Redis connection errors or reservation failures increase.

#### Diagnosis

1. Check gateway health

```bash
curl -s http://localhost:3080/health
```

2. Check events logs

```bash
docker compose logs events --tail=50
```

3. Check Redis logs

```bash
docker compose logs redis
```

#### Fixes

Restart Redis:

```bash
docker compose restart redis
```

Restart Events service:

```bash
docker compose restart events
```

Peer review result: the runbook is clear and helps with fast recovery.

## Final

In this lab I learned:

* how to create alerts in Grafana
* how to write runbooks
* how to investigate incidents
* how to write a blameless postmortem

This lab helped me better understand monitoring and incident response.

Lab 6 — Alerting & Incident Response

============================================================
Task 1 — Alerts, Runbook and Incident Response
============================================================

1. Alert Rules in Grafana

Alert 1: QuickTicket High Error Rate (Critical)

Name:
QuickTicket High Error Rate

PromQL:
sum(rate(gateway_requests_total{status=~"5.."}[5m])) / sum(rate(gateway_requests_total[5m])) * 100 > 10

Condition:
IS ABOVE 10

Evaluate every:
1m

For:
2m (pending)

Labels:
severity=critical

Annotations:
Summary:
High error rate detected: {{ $value | printf "%.2f" }}%

Description:
Gateway is returning too many 5xx errors. Immediate investigation required.


------------------------------------------------------------

Alert 2: QuickTicket SLO Burn Rate (Warning)

Name:
QuickTicket SLO Burn Rate

PromQL:
(1 - (sum(rate(gateway_requests_total{status!~"5.."}[30m])) / sum(rate(gateway_requests_total[30m])))) / (1 - 0.995) > 6

Condition:
IS ABOVE 6

Evaluate every:
1m

For:
5m

Labels:
severity=warning

============================================================
2. Contact Point & Notification Policy
============================================================

Contact Point:
Webhook
https://webhook.site/b395b256-1ac3-4478-8193-ba329270ebf2

Successfully tested:
Received full JSON payload.

Notification Policy:

Default contact point:
quickticket-alerts

Group by:
alertname

Group wait:
30s

Repeat interval:
5m

============================================================
3. Runbook: QuickTicket High Error Rate
============================================================

Alert:
Fires when gateway 5xx error rate exceeds 10% for 2 minutes.

Diagnosis Steps:

1. Check global health:
curl -s http://localhost:3080/health | python3 -m json.tool

2. Check individual services health.

3. Review recent logs:
docker compose logs --tail=100 gateway payments

4. Check Prometheus for payments_requests_total metrics.

Common Causes & Mitigation

Cause:
High PAYMENT_FAILURE_RATE

Identification:
payments health OK, but errors in logs

Resolution:
Set env var to 0.0 and restart Payments


Cause:
Payments container crashed

Identification:
docker compose ps shows not running

Resolution:
docker compose up -d payments


Cause:
Events service down

Identification:
events health check fails

Resolution:
Restart events


Cause:
Database issues

Identification:
Connection errors in events logs

Resolution:
Check postgres and restart events

Escalation:
If unresolved in 10 minutes → notify team lead.

============================================================
4. Incident Simulation & Response
============================================================

Failure Injection:

# High failure rate injection
PAYMENT_FAILURE_RATE=0.8 docker compose -f docker-compose.yaml -f ../docker-compose.monitoring.yaml up -d payments

Timeline

14:52:00
Injected failure (PAYMENT_FAILURE_RATE=0.8)

14:55:12
Alert entered Pending state

14:57:45
Alert Firing (High Error Rate)

14:57:50
Webhook notification received

14:58:10
Started following runbook

14:59:20
Root cause identified (Payments service)

15:00:05
Fix applied (PAYMENT_FAILURE_RATE=0.0)

15:02:30
Alert resolved to Normal

Answer:

From failure injection to alert firing took approximately 5 minutes 45 seconds.

This delay comes from the evaluation interval plus pending period, which is intentional to reduce alert noise but trades off detection speed.

============================================================
5. Proofs
============================================================

• Alert rules created and tested in Grafana.
• Webhook notification received (JSON payload captured).
• Runbook used during a real incident.

============================================================
Task 2 — Blameless Postmortem
============================================================

Postmortem:
High Error Rate Due to Payments Service Degradation

Date:
June 17, 2026

Duration:
10 minutes 30 seconds

Severity:
SEV-3 (Degraded user experience)

Author:
Ravil Khusnutdinov

------------------------------------------------------------
Summary
------------------------------------------------------------

A configuration change increased the payments failure rate to 80%, causing elevated 5xx errors at the gateway level. The issue was detected by our SLO-based alerting and resolved following the runbook.

------------------------------------------------------------
Timeline
------------------------------------------------------------

See the incident timeline above.

------------------------------------------------------------
Root Cause
------------------------------------------------------------

Lack of a dedicated low-level alert on the payments service failure rate allowed the degradation to propagate to the gateway. The existing high error rate alert worked, but the pending period introduced detection delay.

------------------------------------------------------------
What Went Well
------------------------------------------------------------

• Alerting system successfully caught the issue.
• Runbook provided clear diagnostic steps.
• Webhook notification worked immediately.
• Quick recovery once investigation started.

------------------------------------------------------------
What Went Wrong
------------------------------------------------------------

• No service-specific failure rate alert.
• Pending period (2 minutes) delayed detection.
• Runbook did not include direct environment variable inspection.

------------------------------------------------------------
Action Items
------------------------------------------------------------

Action:
Create dedicated alert for PAYMENT_FAILURE_RATE impact

Owner:
Ravil

Priority:
High

Status:
To Do


Action:
Reduce pending period for critical alerts to 90s

Owner:
Ravil

Priority:
High

Status:
To Do


Action:
Enhance runbook with env var and config checks

Owner:
Ravil

Priority:
Medium

Status:
To Do


Action:
Add unit tests for failure rate injection

Owner:
Team

Priority:
Low

Status:
To Do

Most important action item:

Creating a dedicated payments failure rate alert.

Why?

It enables faster detection of isolated service degradation before it affects overall SLOs and user experience.

============================================================
Bonus Task — Cross-Tested Runbook
============================================================

Second Runbook:
Redis Unavailable (Reservations Failing)

Alert:
High Redis error rate or connection failures.

Diagnosis:

1. Check Redis health:
docker compose exec redis redis-cli ping

2. Check gateway/events logs for Redis connection errors.

3. Verify Redis pod status in Kubernetes or Docker.

Fixes:

• Restart Redis container.
• Check network connectivity.
• Scale Redis if under high load.

Cross-testing result:

A classmate successfully diagnosed and fixed the Redis failure using only this runbook in 4 minutes.

Feedback:

"Missing check for Redis memory usage."

Update made:

Added section:
"Check Redis memory usage with docker stats."

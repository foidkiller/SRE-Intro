Lab 6 — Alerting & Incident Response
Task 1 — Alerts, Runbook and Incident Response
1. Alert Rules in Grafana
Alert 1: QuickTicket High Error Rate (Critical)

Name: QuickTicket High Error Rate
PromQL:
promql

sum(rate(gateway_requests_total{status=~"5.."}[5m])) / sum(rate(gateway_requests_total[5m])) * 100
Condition: IS ABOVE 10
Evaluate every: 1m | for: 2m (pending)
Labels: severity=critical
Annotations:
Summary: High error rate detected: {{ $value | printf "%.2f" }}%
Description: Gateway is returning too many 5xx errors. Immediate investigation required.
Alert 2: QuickTicket SLO Burn Rate (Warning)

Name: QuickTicket SLO Burn Rate
PromQL:
promql

(1 - (sum(rate(gateway_requests_total{status!~"5.."}[30m])) / sum(rate(gateway_requests_total[30m])))) / (1 - 0.995)
Condition: IS ABOVE 6
Evaluate every: 1m | for: 5m
Labels: severity=warning
2. Contact Point & Notification Policy
Contact Point: Webhook (https://webhook.site/b395b256-1ac3-4478-8193-ba329270ebf2)
Status: Successfully tested — received full JSON payload.
Notification Policy:

Default contact point: quickticket-alerts
Group by: alertname
Group wait: 30s
Repeat interval: 5m
3. Runbook: QuickTicket High Error Rate
Alert Condition
Fires when gateway 5xx error rate exceeds 10% for 2 minutes.

Diagnosis Steps

Check global health: curl -s http://localhost:3080/health | python3 -m json.tool
Check individual services health status.
Review recent logs: docker compose logs --tail=100 gateway payments
Check Prometheus for payments_requests_total metrics.
Common Causes & Mitigation

Cause	Identification	Resolution
High PAYMENT_FAILURE_RATE	Payments health OK, but errors in logs	Set env var to 0.0 and restart
Payments container crashed	docker compose ps shows not running	docker compose up -d payments
Events service down	Events health check fails	Restart events service
Database issues	Connection errors in events logs	Check postgres and restart events
Escalation: If unresolved in 10 minutes, notify the team lead.

4. Incident Simulation & Response
Failure Injection:

Bash

PAYMENT_FAILURE_RATE=0.8 docker compose -f docker-compose.yaml -f ../docker-compose.monitoring.yaml up -d payments
Timeline:

Time	Event
14:52:00	Injected failure (PAYMENT_FAILURE_RATE=0.8)
14:55:12	Alert entered Pending state
14:57:45	Alert Firing (High Error Rate)
14:57:50	Webhook notification received
14:58:10	Started following runbook
14:59:20	Root cause identified (Payments service)
15:00:05	Fix applied (PAYMENT_FAILURE_RATE=0.0)
15:02:30	Alert resolved to Normal
Response Analysis: From failure injection to alert firing took approximately 5 minutes 45 seconds. This delay is composed of the evaluation interval and the pending period (for: 2m), which is designed to prevent flapping and reduce noise at the cost of detection speed.

5. Proofs
Alert rules created and verified in Grafana.
Webhook notification received (JSON payload captured).
Runbook successfully utilized during the simulated incident.
Task 2 — Blameless Postmortem
Postmortem: High Error Rate Due to Payments Service Degradation

Date: June 17, 2026
Duration: 10 minutes 30 seconds
Severity: SEV-3 (Degraded user experience)
Author: Ravil Khusnutdinov
Summary
A configuration change increased the payments failure rate to 80%, causing elevated 5xx errors at the gateway level. The issue was detected by SLO-based alerting and resolved following the established runbook.

Timeline
(See timeline table in Task 1)

Root Cause
Lack of a dedicated low-level alert on the payments service failure rate allowed the degradation to propagate to the gateway. The existing high error rate alert functioned correctly, but the pending period introduced a delay in detection.

What Went Well

Alerting system successfully captured the incident.
Runbook provided clear, actionable diagnostic steps.
Webhook notification was delivered immediately upon firing.
Recovery was rapid once investigation commenced.
What Went Wrong

Absence of service-specific failure rate alerts.
Pending period (2 minutes) delayed initial detection.
Runbook lacked direct environment variable inspection steps.
Action Items

Action	Owner	Priority
Add alert for payments failures	SRE Team	High
Reduce alert waiting time	SRE Team	High
Improve runbook with env checks	SRE Team	Medium
Add failure scenario tests	Dev Team	Medium
Most important action item: Creating a dedicated payments failure rate alert.
Reasoning: This allows for faster detection of isolated service degradation before it impacts the overall SLOs and user experience.

Bonus Task — Cross-Tested Runbook
Runbook: Redis Unavailable (Reservations Failing)

Alert: High Redis error rate or connection failures.

Diagnosis

Check Redis health: docker compose exec redis redis-cli ping
Check gateway/events logs for Redis connection errors.
Verify Redis container status: docker compose ps redis
Fixes

Restart Redis container: docker compose restart redis
Check network connectivity between services.
Scale Redis or clear cache if memory is exhausted.
Cross-testing Result:
A classmate successfully diagnosed and fixed the Redis failure using this runbook in 4 minutes.
Feedback: "Missing a check for Redis memory usage."
Update: Added section "Check Redis memory usage with docker stats".






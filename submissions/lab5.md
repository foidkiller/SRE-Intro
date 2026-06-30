# Lab 6 — Alerting & Incident Response


## Introduction
In this lab I set up SLO-based alerts in Grafana, simulated an incident, responded to it using my runbook, and wrote a blameless postmortem. It was one of the most interesting labs because I finally got to practice real incident handling.

## Task 1 — Alerts and Incident Response

### Starting the Stack
I launched the full monitoring stack and started generating traffic:

```bash
cd ~/SRE-Intro/app
docker compose -f docker-compose.yaml -f ../docker-compose.monitoring.yaml up -d --build
./loadgen/run.sh 5 600 &
```
Contact Point
Created a Webhook contact point in Grafana using webhook.site. The test notification arrived successfully.
Alert Rules
1. High Error Rate (Critical)

Name: QuickTicket High Error Rate
PromQL: sum(rate(gateway_requests_total{status=~"5.."}[5m])) / sum(rate(gateway_requests_total[5m])) * 100 > 10
Condition: Is Above 10, evaluated every 1m for 2m
Labels: severity=critical

2. SLO Burn Rate (Warning)

Created a second alert for error budget burn rate.

Runbook
Runbook: QuickTicket High Error Rate
Alert

Fires when gateway 5xx error rate exceeds 10% for 2 minutes.

Diagnosis Steps

Check overall health: curl -s http://localhost:3080/health | python3 -m json.tool
Check payments service: curl -s http://localhost:8082/health
Look at logs: docker compose logs payments --tail=50 and gateway logs
Check PAYMENT_FAILURE_RATE environment variable

Common Fixes

Payments service down → restart it
High failure rate → set PAYMENT_FAILURE_RATE=0 and restart
Events issues → restart events service

If not fixed in 10 minutes — escalate to instructor/TA.
Incident Simulation
I simulated a failure by setting PAYMENT_FAILURE_RATE=0.5 in the payments service.
What happened:

Injected the failure
After a few minutes the High Error Rate alert fired
Followed my runbook, found the problem in payments
Fixed it by restoring normal settings
Alert went back to normal

The runbook really helped me stay organized during the incident.
Task 2 — Blameless Postmortem
Postmortem: Payments Service Failure Rate Spike
Summary
During testing I increased the failure rate in the payments service. This caused a rise in 5xx errors on the gateway and triggered our alert. We were able to detect and resolve the issue following the runbook.
Timeline

Failure injected
Alert fired after a few minutes
Started investigation using the runbook
Identified payments service as the root cause
Restored normal operation
Alert resolved

Root Cause
The system did not have enough visibility into the health and failure rate of the payments service. A configuration change in a downstream component was able to affect user-facing error rates before being caught early.
What Went Well

The alert worked and we got notified
The runbook provided useful steps and helped quickly find the problem
We recovered the service fairly fast

What Went Wrong

The pending period delayed detection a bit
No dedicated alert for the payments service itself
Could have had more specific checks for environment variables in the runbook

| Action                          | Owner    | Priority |
| ------------------------------- | -------- | -------- |
| Add alert for payments failures | SRE Team | High     |
| Reduce alert waiting time       | SRE Team | High     |
| Improve runbook                 | SRE Team | Medium   |
| Add failure scenario tests      | Dev Team | Medium   |


The most important action item: Adding a dedicated alert for the payments service.
Why: It would catch the issue much earlier before it starts impacting users and burning the error budget.
Bonus Task — Second Runbook
Runbook: Redis Unavailable

Trigger: Increase in reservation failures or Redis connection errors
Diagnosis: Check gateway health, events logs, and redis logs
Fix: Restart Redis container and then the events service if needed

Tested it myself — the runbook is clear and leads to fast recovery.
Conclusion
This lab gave me good hands-on experience with alerting, incident response, and writing postmortems. I now better understand how important clear runbooks and proper monitoring are in real SRE work.

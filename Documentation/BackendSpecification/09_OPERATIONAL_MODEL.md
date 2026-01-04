# Operational Model (Target)

## 1. Environments
- `local`: developer machine.
- `dev`: integration.
- `staging`: production-like.
- `prod`: production.

## 2. Configuration & Secrets
- Config via environment variables and config files.
- Secrets in secret manager (Vault/KMS/Cloud secrets), not in repo.

## 3. Deployment
- Blue/green or rolling deployments.
- DB migrations executed as a gated step.

## 4. Data Migrations
- Forward-only migrations.
- Backward compatible changes first (expand/contract).

## 5. Scheduled Jobs
- Ranking recomputation windows.
- Notification digests.
- Payment reconciliation.
- Cleanup jobs for expired tokens and old events.

- AI automation workers (continuous):
	- Outbox publisher (event_outbox → bus)
	- AI review executor (`ai_job` type REVIEW)
	- Translation executor (`ai_job` type TRANSLATE)
	- Queue watchdog (alerts on lag/backlog)

## 6. Observability
- Logs: JSON structured.
- Metrics: Prometheus format.
- Tracing: OpenTelemetry.
- Dashboards: latency, error rate, saturation, queue lag.

## 7. Incident Response
- Severity levels and paging.
- Runbooks: payment outage, DB saturation, cache outage.

- Runbooks (AI automation):
	- AI provider outage (timeouts/errors): stop retries temporarily, raise alert, keep submissions pending, communicate degraded publish SLA.
	- AI queue lag spike: scale workers, reduce concurrency, enforce stricter quotas, prioritize by risk.
	- Approval backlog: enable queue filtering by risk, assign moderators, adjust auto-approve thresholds (admin).

## 8. Backups & Retention
- PostgreSQL: base backups + WAL.
- Redis: persistence optional; treat as cache.
- Retention: align with compliance and cost.

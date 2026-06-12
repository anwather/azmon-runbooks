# SQL Server — SQL Agent job last run failed

**Alert ID:** `sql-agent-job-last-run`
**Severity:** 3 Warning
**Category:** Agent
**Detection mechanism:** DMV probe
**Source SCOM monitor / rule (if any):** `Monitor.AgentJob.LastRunState`

---

## Symptom

A discovered SQL Agent job has a most recent outcome of failed.

## What it means

The last scheduled or manual execution failed. Business impact depends on the job: backups can be missed, ETL may stall, replication/log-shipping can lag, or maintenance may not complete.

## Common causes

- Job step returned non-zero error
- Proxy credential or subsystem permission failure
- Deadlock/blocking or timeout in T-SQL step
- Missing file/share/package for CmdExec/SSIS step
- Backup/replication/log-shipping dependency failed

## How to diagnose

Start with the 24-hour Log Analytics query to identify the affected computer, instance, database, replica, event, and payload. Then connect to the affected SQL Server instance with a DBA account and run the T-SQL checks shown below.

Record these triage facts before changing anything:

- Alert fire time, first-seen time, and whether this is recurring.
- Affected computer, SQL instance, database, AG, replica, file, job, or error number from the payload.
- Current SQL Server build from `SELECT @@VERSION` and any recent CU, OS, storage, or failover change.
- Whether the condition aligns with an approved maintenance window, failover test, restore, or deployment.
- Whether the same host also has storage, cluster, service, or Agent alerts in the last 24 hours.
- Whether user impact is active: failed connections, failed writes, stale secondary reads, missed backups, or job backlog.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.sql"
| where EventID in (7270, 7570, 7870)
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| extend Severity = tostring(P.severity), Check = tostring(P.check), Message = tostring(P.message), Payload = P.payload
| extend InstanceName = tostring(P.payload.instanceName), DatabaseName = tostring(P.payload.databaseName)
| project TimeGenerated, Computer, InstanceName, DatabaseName, Severity, Check, Message, Payload
| order by TimeGenerated desc
```

```sql
-- T-SQL the DBA runs on the affected instance
SELECT TOP (20) j.name, h.run_date, h.run_time, h.run_duration, h.run_status, h.message
FROM msdb.dbo.sysjobs j
JOIN msdb.dbo.sysjobhistory h ON j.job_id = h.job_id
WHERE h.step_id = 0
ORDER BY msdb.dbo.agent_datetime(h.run_date, h.run_time) DESC;

SELECT j.name, a.start_execution_date, a.stop_execution_date,
       DATEDIFF(minute, a.start_execution_date, COALESCE(a.stop_execution_date, SYSDATETIME())) AS elapsed_min
FROM msdb.dbo.sysjobactivity a
JOIN msdb.dbo.sysjobs j ON a.job_id = j.job_id
WHERE a.start_execution_date IS NOT NULL AND a.stop_execution_date IS NULL;
```

## How to fix

1. Open the exact SQL Agent job history row and failed step output; fix the failing step rather than rerunning blindly.
2. Correct proxy credential, subsystem, file path, package, or linked-server dependency identified in the step message.
3. For T-SQL steps, diagnose blocking/waits with `sys.dm_exec_requests` and fix the root query or blocking session.
4. Rerun the job manually only after dependencies are healthy; success is a new `sysjobhistory` outcome of Succeeded.
5. For duration alerts, tune per-job thresholds only after confirming runtime is normal for current data volume.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to the DBA team if the alert persists after the listed fixes, affects multiple instances/databases, requires downtime, or involves data loss, security key material, cluster failover, or vendor support.

## Related alerts

- [sql-agent-service-status](./sql-agent-service-status.md)
- [sql-agent-unable-to-connect](./sql-agent-unable-to-connect.md)
- [sql-agent-job-failed](./sql-agent-job-failed.md)
- [sql-agent-job-duration](./sql-agent-job-duration.md)

## References

- KQL: `packs/sql/docs/kql/probe-agent-job-last-run.kql`
- Bicep: `packs/sql/alerts/probes.bicep`
- Probe script: `scripts/packs/sql/Public/Test-SqlAgentJobs.ps1`
- Threshold defaults: `packs/sql/parameters/thresholds.example.json`
- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Microsoft docs: https://learn.microsoft.com/sql/
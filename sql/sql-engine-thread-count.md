# SQL Server — Free SQL worker thread count is too low

**Alert ID:** `sql-engine-thread-count`
**Severity:** 2 Error
**Category:** Performance
**Detection mechanism:** DMV probe
**Source SCOM monitor / rule (if any):** `Monitor.DBEngine.ThreadCount`

---

## Symptom

The number of available worker threads is below threshold.

## What it means

Worker starvation is possible. New requests can queue even when CPU is not maxed, usually because blocking, parallelism, long-running external waits, or too many sessions consume workers.

## Common causes

- Blocking chain holding many workers
- Parallel queries consuming workers per request
- External waits such as linked servers or network I/O
- Too many concurrent sessions
- Worker leak/product issue requiring CU

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
| where EventID in (7250, 7550, 7850)
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
SELECT max_workers_count, scheduler_count, cpu_count, committed_kb, committed_target_kb
FROM sys.dm_os_sys_info;

SELECT scheduler_id, current_workers_count, active_workers_count,
       runnable_tasks_count, work_queue_count, pending_disk_io_count
FROM sys.dm_os_schedulers
WHERE status = 'VISIBLE ONLINE'
ORDER BY runnable_tasks_count DESC;
```

## How to fix

1. Identify the dominant wait, memory clerk, scheduler, or request from the diagnostic query before changing instance settings.
2. Kill or tune only the confirmed offending request; capture query text and plan first if possible.
3. Fix blocking chains, missing indexes/statistics, parameter-sensitive plans, or excessive MAXDOP/cost threshold issues as indicated.
4. For memory pressure, right-size max server memory and Resource Governor pools; do not starve the OS or monitoring agents.
5. Success is sustained drop in waits/CPU/memory pressure and probe rows returning OK for at least two intervals.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to the DBA team if the alert persists after the listed fixes, affects multiple instances/databases, requires downtime, or involves data loss, security key material, cluster failover, or vendor support.

## Related alerts

- [sql-engine-cpu-utilization](./sql-engine-cpu-utilization.md)
- [sql-engine-stolen-server-memory](./sql-engine-stolen-server-memory.md)
- [sql-engine-average-wait-time](./sql-engine-average-wait-time.md)

## References

- KQL: `packs/sql/docs/kql/probe-thread-count.kql`
- Bicep: `packs/sql/alerts/probes.bicep`
- Probe script: `scripts/packs/sql/Public/Test-SqlThreadCount.ps1`
- Threshold defaults: `packs/sql/parameters/thresholds.example.json`
- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Microsoft docs: https://learn.microsoft.com/sql/
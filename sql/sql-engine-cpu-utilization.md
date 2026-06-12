# SQL Server — SQL Server CPU utilization is too high

**Alert ID:** `sql-engine-cpu-utilization`
**Severity:** 2 Error
**Category:** Performance
**Detection mechanism:** Perf counter
**Source SCOM monitor / rule (if any):** `Monitor.DBEngine.CPUUtilization`

---

## Symptom

The SQL Server process CPU utilization exceeded threshold.

## What it means

Schedulers are saturated or close to saturation. User queries, recompilation, parallelism, CLR, or external pressure can increase latency and cause connection timeouts.

## Common causes

- Expensive ad hoc or parameter-sensitive query plan
- Missing/stale statistics causing scans
- Excessive parallelism or recompilation
- ETL/CHECKDB/index maintenance window overlap
- External CPU pressure on the VM host

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
Perf
| where TimeGenerated > ago(24h)
| where CounterName =~ "% Processor Time" or InstanceName has "sqlservr"
| summarize MaxValue=max(CounterValue), AvgValue=avg(CounterValue) by bin(TimeGenerated, 15m), Computer, ObjectName, CounterName, InstanceName
| where MaxValue > 0
| order by TimeGenerated desc
```

```sql
-- T-SQL the DBA runs on the affected instance
SELECT TOP (20) wait_type, waiting_tasks_count, wait_time_ms, signal_wait_time_ms
FROM sys.dm_os_wait_stats
WHERE wait_type NOT LIKE 'SLEEP%' AND wait_type NOT LIKE 'BROKER%'
ORDER BY wait_time_ms DESC;

SELECT TOP (20) r.session_id, r.status, r.cpu_time, r.total_elapsed_time,
       r.wait_type, r.blocking_session_id, SUBSTRING(t.text,1,4000) AS sql_text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
ORDER BY r.cpu_time DESC;
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

- [sql-engine-stolen-server-memory](./sql-engine-stolen-server-memory.md)
- [sql-engine-thread-count](./sql-engine-thread-count.md)
- [sql-engine-average-wait-time](./sql-engine-average-wait-time.md)

## References

- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Raw research: `docs/research/sql/raw/agent-b-unit-monitors.md` (for monitor IDs)
- Microsoft docs: https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-os-performance-counters-transact-sql
- Microsoft docs: https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-os-wait-stats-transact-sql
- Kevin Holman: https://kevinholman.com (search for the SCOM monitor name when relevant)

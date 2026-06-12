# SQL Server — User resource pool memory consumption is too high

**Alert ID:** `sql-user-resource-pool-memory`
**Severity:** 2 Error
**Category:** Performance
**Detection mechanism:** DMV probe
**Source SCOM monitor / rule (if any):** `Monitor.UserResourcePool.MemoryConsumption`

---

## Symptom

A Resource Governor user pool is using too much of its target memory.

## What it means

Queries assigned to the pool may experience memory grant waits, spills, or failures. In-Memory OLTP pools can reject allocations when memory pressure is severe.

## Common causes

- Resource Governor classifier overloaded pool
- Memory grant-heavy queries in pool
- Target/max memory percentage too low
- In-Memory OLTP data growth
- Stats/plan regression causing spills

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
| where EventID in (7120, 7420, 7720)
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
SELECT TOP (20) type, name, pages_kb, virtual_memory_committed_kb, awe_allocated_kb
FROM sys.dm_os_memory_clerks
ORDER BY pages_kb DESC;

SELECT name, min_memory_percent, max_memory_percent, used_memory_kb, target_memory_kb,
       CAST(used_memory_kb * 100.0 / NULLIF(target_memory_kb,0) AS decimal(5,2)) AS used_pct
FROM sys.dm_resource_governor_resource_pools;
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
- [sql-engine-thread-count](./sql-engine-thread-count.md)
- [sql-engine-average-wait-time](./sql-engine-average-wait-time.md)

## References

- KQL: `packs/sql/docs/kql/probe-resource-pool-memory.kql`
- Bicep: `packs/sql/alerts/probes.bicep`
- Probe script: `scripts/packs/sql/Public/Test-SqlResourcePoolMemory.ps1`
- Threshold defaults: `packs/sql/parameters/thresholds.example.json`
- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Microsoft docs: https://learn.microsoft.com/sql/
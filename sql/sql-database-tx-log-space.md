# SQL Server — Transaction log free space is critical

**Alert ID:** `sql-database-tx-log-space`
**Severity:** 2 Error
**Category:** Storage
**Detection mechanism:** DMV probe
**Source SCOM monitor / rule (if any):** `Monitor.Database.TransactionLogSpaceFreePercent`

---

## Symptom

A database transaction log has too little free space remaining.

## What it means

The log cannot truncate or grow enough for current workload. Transactions may fail with error 9002 and recovery, AG, replication, or log shipping may lag.

## Common causes

- Log backups not running in FULL/BULK_LOGGED recovery
- Long transaction preventing truncation
- AG, replication, CDC, or log shipping holding log reuse
- Log file max_size or volume full
- Index maintenance generated large log volume

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
| where EventID in (7100, 7400, 7700)
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
DBCC SQLPERF(LOGSPACE);

SELECT name, recovery_model_desc, log_reuse_wait_desc
FROM sys.databases
ORDER BY log_reuse_wait_desc, name;

SELECT DB_NAME(database_id) AS database_name, total_log_size_in_bytes/1048576 AS total_log_mb,
       used_log_space_in_bytes/1048576 AS used_log_mb, used_log_space_in_percent
FROM sys.dm_db_log_space_usage;
```

```powershell
Get-Counter '\PhysicalDisk(*)\Avg. Disk sec/Read','\PhysicalDisk(*)\Avg. Disk sec/Write' -SampleInterval 5 -MaxSamples 3
Get-Volume | Select-Object DriveLetter, FileSystemLabel, SizeRemaining, Size
Get-EventLog -LogName System -EntryType Error -Newest 50 | Where-Object {$_.Source -match 'disk|stor|ntfs'}
```

## How to fix

1. Identify `log_reuse_wait_desc` before taking action; it tells you why truncation is blocked.
2. If FULL/BULK_LOGGED and backups are overdue, take a log backup to durable storage: `BACKUP LOG [db] TO DISK = N<path>.trn WITH CHECKSUM;`.
3. If a long transaction blocks reuse, work with the application owner before killing the session; success is log reuse wait clearing.
4. If the log cannot grow, add disk or grow the file with `ALTER DATABASE [db] MODIFY FILE`.
5. Do not switch to SIMPLE recovery unless the DBA accepts breaking the log backup chain and immediately takes a new full backup afterward.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to DBA and storage teams if growth requires emergency capacity, if log truncation is blocked by AG/replication/CDC for more than one business cycle, or if transactions are failing.

## Related alerts

- [sql-database-rows-size-percent](./sql-database-rows-size-percent.md)
- [sql-error-9002-log-full](./sql-error-9002-log-full.md)
- [sql-error-823-io-logical-error](./sql-error-823-io-logical-error.md)

## References

- KQL: `packs/sql/docs/kql/probe-log-space.kql`
- Bicep: `packs/sql/alerts/probes.bicep`
- Probe script: `scripts/packs/sql/Public/Test-SqlLogSpace.ps1`
- Threshold defaults: `packs/sql/parameters/thresholds.example.json`
- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Microsoft docs: https://learn.microsoft.com/sql/
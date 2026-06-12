# SQL Server — SQL error 9002 transaction log full

**Alert ID:** `sql-error-9002-log-full`
**Severity:** 2 Error
**Category:** Storage
**Detection mechanism:** Custom Text Logs (ERRORLOG) DCR
**Source SCOM monitor / rule (if any):** `EventRule.DBEngine.Database_log_file_is_full._Back_up_the_transaction_log_for_the_database_to_free_up_some_log_space_1_5_Rule`

---

## Symptom

The proposed alert fires when SQL ERRORLOG ingestion sees error 9002. Typical message text: "The transaction log for database is full due to a log reuse wait."

## What it means

The transaction log cannot reuse or grow enough space. Write transactions can fail and recovery-dependent features such as AGs and log shipping may stall.

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
SqlErrorLog_CL
| where TimeGenerated > ago(24h)
| extend ErrorNumber = toint(coalesce(ErrorNumber, ErrorNumber_s, extract(@"Error:\s*(\d+)", 1, RawData)))
| where ErrorNumber == 9002
| extend InstanceName = tostring(coalesce(InstanceName_s, InstanceName))
| extend DatabaseName = tostring(coalesce(DatabaseName_s, DatabaseName))
| project TimeGenerated, Computer, InstanceName, DatabaseName, ErrorNumber,
          Severity=tostring(coalesce(Severity_s, Severity)), Message=tostring(coalesce(Message, RawData))
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
- [sql-database-tx-log-space](./sql-database-tx-log-space.md)
- [sql-error-823-io-logical-error](./sql-error-823-io-logical-error.md)

## References

- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Raw research: `docs/research/sql/raw/agent-c-rules-part1.md` / `agent-c-rules-part2.md` (for rule IDs)
- Microsoft docs: https://learn.microsoft.com/sql/relational-databases/errors-events/mssqlserver-9002-database-engine-error
- Kevin Holman: https://kevinholman.com (search for the SCOM monitor name when relevant)

# SQL Server — SQL error 1105 filegroup is full

**Alert ID:** `sql-error-1105-disk-full`
**Severity:** 2 Error
**Category:** Storage
**Detection mechanism:** Custom Text Logs (ERRORLOG) DCR
**Source SCOM monitor / rule (if any):** `EventRule.DBEngine.Could_not_allocate_space_for_object__in_database__because_the__filegroup_is_full_1_5_Rule`

---

## Symptom

The proposed alert fires when SQL ERRORLOG ingestion sees error 1105. Typical message text: "Could not allocate space for object in database because the filegroup is full."

## What it means

SQL could not allocate database pages. Inserts, index maintenance, tempdb worktables, version store, or autogrowth may fail until space is added.

## Common causes

- Data file max_size reached
- Volume hosting data files is nearly full
- Autogrowth disabled or too small
- Unexpected data load or index rebuild growth
- Filegroup not balanced across files

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
| where ErrorNumber == 1105
| extend InstanceName = tostring(coalesce(InstanceName_s, InstanceName))
| extend DatabaseName = tostring(coalesce(DatabaseName_s, DatabaseName))
| project TimeGenerated, Computer, InstanceName, DatabaseName, ErrorNumber,
          Severity=tostring(coalesce(Severity_s, Severity)), Message=tostring(coalesce(Message, RawData))
| order by TimeGenerated desc
```

```sql
-- T-SQL the DBA runs on the affected instance
SELECT DB_NAME(database_id) AS database_name, type_desc, name, physical_name,
       size * 8 / 1024 AS size_mb, max_size, growth, is_percent_growth
FROM sys.master_files
ORDER BY database_name, type_desc, file_id;

SELECT DB_NAME(mf.database_id) AS database_name, mf.name, vs.volume_mount_point,
       vs.available_bytes / 1048576 AS volume_free_mb, vs.total_bytes / 1048576 AS volume_total_mb
FROM sys.master_files mf
CROSS APPLY sys.dm_os_volume_stats(mf.database_id, mf.file_id) vs;
```

```powershell
Get-Counter '\PhysicalDisk(*)\Avg. Disk sec/Read','\PhysicalDisk(*)\Avg. Disk sec/Write' -SampleInterval 5 -MaxSamples 3
Get-Volume | Select-Object DriveLetter, FileSystemLabel, SizeRemaining, Size
Get-EventLog -LogName System -EntryType Error -Newest 50 | Where-Object {$_.Source -match 'disk|stor|ntfs'}
```

## How to fix

1. Free or add space on the hosting volume before changing SQL files.
2. Enable or increase autogrowth with a fixed MB increment: `ALTER DATABASE [db] MODIFY FILE (NAME = Nfile, FILEGROWTH = 1024MB);`.
3. Add a data file to the full filegroup: `ALTER DATABASE [db] ADD FILE (NAME=Ndb_data2, FILENAME=N<path>, SIZE=8192MB, FILEGROWTH=1024MB) TO FILEGROUP [fg];`.
4. Remove unneeded data only through application-approved purge/archive procedures; do not shrink as a primary fix.
5. After growth, rerun the file/volume queries and confirm free percent and absolute MB are above thresholds.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to DBA and storage teams if growth requires emergency capacity, if log truncation is blocked by AG/replication/CDC for more than one business cycle, or if transactions are failing.

## Related alerts

- [sql-database-rows-size-percent](./sql-database-rows-size-percent.md)
- [sql-database-tx-log-space](./sql-database-tx-log-space.md)
- [sql-error-9002-log-full](./sql-error-9002-log-full.md)
- [sql-error-823-io-logical-error](./sql-error-823-io-logical-error.md)

## References

- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Raw research: `docs/research/sql/raw/agent-c-rules-part1.md` / `agent-c-rules-part2.md` (for rule IDs)
- Microsoft docs: https://learn.microsoft.com/sql/relational-databases/errors-events/mssqlserver-1105-database-engine-error
- Kevin Holman: https://kevinholman.com (search for the SCOM monitor name when relevant)

# SQL Server — DB Engine disk read latency is high

**Alert ID:** `sql-engine-disk-latency-read`
**Severity:** 2 Error
**Category:** Storage
**Detection mechanism:** Perf counter
**Source SCOM monitor / rule (if any):** `Monitor.DBEngine.DiskLatency`

---

## Symptom

Average SQL volume read latency is above the proposed threshold for multiple samples.

## What it means

Data and log reads are taking too long at the Windows storage layer. Queries may stall on PAGEIOLATCH waits, AG redo may fall behind, and backups or CHECKDB may run much longer.

## Common causes

- Storage latency or throttling on data/log volume
- SAN/virtual disk queue depth saturation
- Antivirus or backup filter driver delay
- Tempdb or CHECKDB scanning large files
- Underlying disk path failover or degraded RAID

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
| where CounterName =~ "Avg. Disk sec/Read" or InstanceName has "sqlservr"
| summarize MaxValue=max(CounterValue), AvgValue=avg(CounterValue) by bin(TimeGenerated, 15m), Computer, ObjectName, CounterName, InstanceName
| where MaxValue > 0
| order by TimeGenerated desc
```

```sql
-- T-SQL the DBA runs on the affected instance
SELECT DB_NAME(vfs.database_id) AS database_name, mf.physical_name,
       vfs.num_of_reads, vfs.io_stall_read_ms, vfs.num_of_writes, vfs.io_stall_write_ms,
       CAST(vfs.io_stall_read_ms / NULLIF(vfs.num_of_reads,0) AS decimal(18,2)) AS avg_read_ms,
       CAST(vfs.io_stall_write_ms / NULLIF(vfs.num_of_writes,0) AS decimal(18,2)) AS avg_write_ms
FROM sys.dm_io_virtual_file_stats(NULL, NULL) vfs
JOIN sys.master_files mf ON vfs.database_id = mf.database_id AND vfs.file_id = mf.file_id
ORDER BY COALESCE(avg_read_ms,0) + COALESCE(avg_write_ms,0) DESC;

SELECT database_id, file_id, page_id, event_type, error_count, last_update_date
FROM msdb.dbo.suspect_pages
ORDER BY last_update_date DESC;
```

```powershell
Get-Counter '\PhysicalDisk(*)\Avg. Disk sec/Read','\PhysicalDisk(*)\Avg. Disk sec/Write' -SampleInterval 5 -MaxSamples 3
Get-Volume | Select-Object DriveLetter, FileSystemLabel, SizeRemaining, Size
Get-EventLog -LogName System -EntryType Error -Newest 50 | Where-Object {$_.Source -match 'disk|stor|ntfs'}
```

## How to fix

1. Freeze risky maintenance such as index rebuilds until storage health is understood.
2. Run `DBCC CHECKDB([database]) WITH NO_INFOMSGS, ALL_ERRORMSGS` for affected databases and capture full output.
3. Check Windows System log and storage array/VM disk health for matching disk, storport, NTFS, or path failover errors.
4. If CHECKDB is clean but 823/824/825 persists, move files to healthy storage or engage storage vendor; do not ignore recurring 825 warnings.
5. If CHECKDB reports corruption, restore clean pages/database from backup where possible; prefer restore over repair.
6. Use `REPAIR_ALLOW_DATA_LOSS` only after DBA approval, verified backups, and business sign-off because it can delete data.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to the DBA team if the alert persists after the listed fixes, affects multiple instances/databases, requires downtime, or involves data loss, security key material, cluster failover, or vendor support.

## Related alerts

- [sql-database-rows-size-percent](./sql-database-rows-size-percent.md)
- [sql-database-tx-log-space](./sql-database-tx-log-space.md)
- [sql-error-9002-log-full](./sql-error-9002-log-full.md)
- [sql-error-823-io-logical-error](./sql-error-823-io-logical-error.md)

## References

- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Raw research: `docs/research/sql/raw/agent-b-unit-monitors.md` (for monitor IDs)
- Microsoft docs: https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-io-virtual-file-stats-transact-sql
- Microsoft docs: https://learn.microsoft.com/sql/relational-databases/databases/display-data-and-log-space-information-for-a-database
- Kevin Holman: https://kevinholman.com (search for the SCOM monitor name when relevant)

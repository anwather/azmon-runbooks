# SQL Server — Database ROWS data free space is low

**Alert ID:** `sql-database-rows-size-percent`
**Severity:** 3 Warning
**Category:** Storage
**Detection mechanism:** DMV probe
**Source SCOM monitor / rule (if any):** `Monitor.Database.RowsSizePercent`

---

## Symptom

A database ROWS file/filegroup has less free space than the configured percent and absolute MB thresholds.

## What it means

Data files or their hosting volumes are running out of usable space. Inserts, index maintenance, and autogrowth can fail or stall.

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
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.sql"
| where EventID in (7290, 7590, 7890)
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

- [sql-database-tx-log-space](./sql-database-tx-log-space.md)
- [sql-error-9002-log-full](./sql-error-9002-log-full.md)
- [sql-error-823-io-logical-error](./sql-error-823-io-logical-error.md)

## References

- KQL: `packs/sql/docs/kql/probe-data-file-space.kql`
- Bicep: `packs/sql/alerts/probes.bicep`
- Probe script: `scripts/packs/sql/Public/Test-SqlDataFileSpace.ps1`
- Threshold defaults: `packs/sql/parameters/thresholds.example.json`
- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Microsoft docs: https://learn.microsoft.com/sql/
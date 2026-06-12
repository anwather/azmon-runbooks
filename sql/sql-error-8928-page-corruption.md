# SQL Server — SQL error 8928 DBCC allocation/page error

**Alert ID:** `sql-error-8928-page-corruption`
**Severity:** 1 Critical
**Category:** ERRORLOG
**Detection mechanism:** Custom Text Logs (ERRORLOG) DCR
**Source SCOM monitor / rule (if any):** `EventRule.DBEngine.DBCC_executed_found_and_not_repaired_errors_Rule`

---

## Symptom

The proposed alert fires when SQL ERRORLOG ingestion sees error 8928. Typical message text: "Object ID, index ID, partition ID, alloc unit ID: Page could not be processed. See other errors."

## What it means

DBCC or SQL detected physical/logical consistency errors. Data may be corrupt and repair choices can cause data loss; restore planning is required.

## Common causes

- Physical page corruption from storage
- Allocation metadata inconsistency
- Prior repair/restore left inconsistent objects
- Memory/storage subsystem returning stale data
- DBCC found errors but could not repair online

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
| where ErrorNumber == 8928
| extend InstanceName = tostring(coalesce(InstanceName_s, InstanceName))
| extend DatabaseName = tostring(coalesce(DatabaseName_s, DatabaseName))
| project TimeGenerated, Computer, InstanceName, DatabaseName, ErrorNumber,
          Severity=tostring(coalesce(Severity_s, Severity)), Message=tostring(coalesce(Message, RawData))
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

Escalate immediately to DBA leadership and storage/vendor support for repeated 823/824/832 or any CHECKDB allocation/consistency error. If CHECKDB lists `repair_allow_data_loss` as the only repair option, do not run repair until restore options and business approval are documented.

## Related alerts

- [sql-engine-service-status](./sql-engine-service-status.md)
- [sql-engine-status](./sql-engine-status.md)
- [sql-engine-wmi-health](./sql-engine-wmi-health.md)

## References

- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Raw research: `docs/research/sql/raw/agent-c-rules-part1.md` / `agent-c-rules-part2.md` (for rule IDs)
- Microsoft docs: https://learn.microsoft.com/sql/relational-databases/errors-events/mssqlserver-8928-database-engine-error
- Kevin Holman: https://kevinholman.com (search for the SCOM monitor name when relevant)

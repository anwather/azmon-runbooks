# SQL Server — SQL error 3201 cannot open backup device

**Alert ID:** `sql-error-3201-cannot-open-backup-device`
**Severity:** 2 Error
**Category:** ERRORLOG
**Detection mechanism:** Custom Text Logs (ERRORLOG) DCR
**Source SCOM monitor / rule (if any):** `EventRule.DBEngine.Cannot_open_backup_device.__1_5_Rule`

---

## Symptom

The proposed alert fires when SQL ERRORLOG ingestion sees error 3201. Typical message text: "Cannot open backup device. Operating system error details follow in the error log."

## What it means

A backup or restore operation could not access its media or complete. The backup chain may have a gap and restore objectives are at risk.

## Common causes

- Backup path missing, inaccessible, or out of space
- SQL service account lacks share or NTFS rights
- Backup target device timed out or disconnected
- Previous media set corrupt or wrong append/overwrite option
- VSS/third-party backup tool conflict

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
| where ErrorNumber == 3201
| extend InstanceName = tostring(coalesce(InstanceName_s, InstanceName))
| extend DatabaseName = tostring(coalesce(DatabaseName_s, DatabaseName))
| project TimeGenerated, Computer, InstanceName, DatabaseName, ErrorNumber,
          Severity=tostring(coalesce(Severity_s, Severity)), Message=tostring(coalesce(Message, RawData))
| order by TimeGenerated desc
```

```sql
-- T-SQL the DBA runs on the affected instance
SELECT TOP (50) database_name, type, backup_start_date, backup_finish_date,
       is_copy_only, has_backup_checksums, bmf.physical_device_name
FROM msdb.dbo.backupset bs
LEFT JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
ORDER BY backup_finish_date DESC;

EXEC master.dbo.xp_fixeddrives;
```

## How to fix

1. Verify the target path exists and SQL Server service account has write/read permission.
2. Free space or choose a different backup location; use backup compression where appropriate.
3. Run a manual backup with `WITH CHECKSUM, STATS = 5` and capture OS error details if it fails.
4. If media is corrupt, start a new media set and validate restore with `RESTORE VERIFYONLY WITH CHECKSUM`.
5. After successful backup, confirm backup chain continuity and update maintenance job output paths.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to DBA/storage owners if backup chain continuity is broken, restore validation fails, media errors repeat, or RPO/RTO commitments are at risk.

## Related alerts

- [sql-engine-service-status](./sql-engine-service-status.md)
- [sql-engine-status](./sql-engine-status.md)
- [sql-engine-wmi-health](./sql-engine-wmi-health.md)

## References

- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Raw research: `docs/research/sql/raw/agent-c-rules-part1.md` / `agent-c-rules-part2.md` (for rule IDs)
- Microsoft docs: https://learn.microsoft.com/sql/relational-databases/errors-events/mssqlserver-3201-database-engine-error
- Kevin Holman: https://kevinholman.com (search for the SCOM monitor name when relevant)

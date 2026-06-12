# SQL Server — Database is not in an expected online state

**Alert ID:** `sql-database-status`
**Severity:** 2 Error
**Category:** Database
**Detection mechanism:** DMV probe
**Source SCOM monitor / rule (if any):** `Monitor.Database.DBStatus`

---

## Symptom

A database is OFFLINE, SUSPECT, EMERGENCY, RECOVERY_PENDING, or otherwise outside the expected state.

## What it means

SQL cannot make the database normally available. User connections fail or see read-only/standby behavior depending on state; AG or log-shipping destinations may intentionally be RESTORING/STANDBY.

## Common causes

- Database set OFFLINE or SINGLE_USER for maintenance
- Recovery pending due to missing/inaccessible files
- SUSPECT from I/O or log corruption
- RESTORING/STANDBY expected for log shipping secondary
- AG database not joined or not recovered

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
| where EventID in (7230, 7530, 7830)
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
SELECT name, state_desc, user_access_desc, is_read_only, is_in_standby,
       recovery_model_desc, log_reuse_wait_desc
FROM sys.databases
ORDER BY CASE WHEN state_desc = 'ONLINE' THEN 1 ELSE 0 END, name;

SELECT DB_NAME(database_id) AS database_name, file_id, type_desc, physical_name, state_desc
FROM sys.master_files
ORDER BY database_name, file_id;
```

## How to fix

1. Confirm whether the state is expected for maintenance, restore, log shipping, or AG seeding.
2. For RECOVERY_PENDING/SUSPECT, fix missing files, storage, or permissions before attempting recovery.
3. Run `DBCC CHECKDB([db]) WITH NO_INFOMSGS, ALL_ERRORMSGS` once the database can be accessed safely.
4. Restore from known-good backups if corruption or inaccessible files cannot be repaired cleanly.
5. Bring intentional offline databases online with `ALTER DATABASE [db] SET ONLINE` only after owner approval.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to the DBA team if the alert persists after the listed fixes, affects multiple instances/databases, requires downtime, or involves data loss, security key material, cluster failover, or vendor support.

## Related alerts

- [sql-engine-service-status](./sql-engine-service-status.md)
- [sql-engine-status](./sql-engine-status.md)
- [sql-engine-wmi-health](./sql-engine-wmi-health.md)

## References

- KQL: `packs/sql/docs/kql/probe-database-status.kql`
- Bicep: `packs/sql/alerts/probes.bicep`
- Probe script: `scripts/packs/sql/Public/Test-SqlDatabaseStatus.ps1`
- Threshold defaults: `packs/sql/parameters/thresholds.example.json`
- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Microsoft docs: https://learn.microsoft.com/sql/
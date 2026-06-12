# SQL Server — Log shipping is out of sync

**Alert ID:** `sql-database-log-shipping`
**Severity:** 2 Error
**Category:** Database
**Detection mechanism:** DMV probe
**Source SCOM monitor / rule (if any):** `Monitor.Database.LogShipping`

---

## Symptom

A log shipping primary has not backed up, or a secondary has not restored, within the configured threshold.

## What it means

The log-shipping restore chain is delayed or broken. Recovery-point objectives are at risk and the secondary may be unusable for DR until jobs catch up.

## Common causes

- Backup/copy/restore Agent job failed
- Network share permission or path unavailable
- Secondary restore blocked by users
- Primary log backups taken outside LS chain
- Time skew or threshold too low

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
| where EventID in (7110, 7410, 7710)
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
SELECT primary_database, last_backup_file, last_backup_date,
       backup_threshold, threshold_alert, threshold_alert_enabled
FROM msdb.dbo.log_shipping_monitor_primary;

SELECT secondary_database, last_copied_file, last_copied_date,
       last_restored_file, last_restored_date, restore_threshold
FROM msdb.dbo.log_shipping_monitor_secondary;
```

## How to fix

1. Fix the first failing SQL Agent job in the backup-copy-restore chain.
2. Repair share permissions and path access for SQL service/Agent proxy accounts.
3. If secondary users block restore, disconnect sessions or restore WITH STANDBY according to DR runbook.
4. Take/copy/restore missing log backups in sequence; reinitialize only if the chain is broken.
5. Success is primary and secondary monitor tables showing current last backup/copy/restore times within threshold.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to DBA/storage owners if backup chain continuity is broken, restore validation fails, media errors repeat, or RPO/RTO commitments are at risk.

## Related alerts

- [sql-engine-service-status](./sql-engine-service-status.md)
- [sql-engine-status](./sql-engine-status.md)
- [sql-engine-wmi-health](./sql-engine-wmi-health.md)

## References

- KQL: `packs/sql/docs/kql/probe-log-shipping.kql`
- Bicep: `packs/sql/alerts/probes.bicep`
- Probe script: `scripts/packs/sql/Public/Test-SqlLogShipping.ps1`
- Threshold defaults: `packs/sql/parameters/thresholds.example.json`
- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Microsoft docs: https://learn.microsoft.com/sql/
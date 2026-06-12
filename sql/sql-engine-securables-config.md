# SQL Server — Instance securables are inaccessible

**Alert ID:** `sql-engine-securables-config`
**Severity:** 3 Warning
**Category:** Security
**Detection mechanism:** DMV probe
**Source SCOM monitor / rule (if any):** `DBEngine.Configuration.SecurablesConfigMonitor`

---

## Symptom

The monitoring login cannot read required instance-level catalog or DMV securables.

## What it means

The probe is not seeing enough metadata to reliably monitor SQL. This is usually a permissions or ownership-chain issue, not a database outage.

## Common causes

- Monitoring login lacks VIEW SERVER STATE/VIEW ANY DEFINITION
- Explicit DENY on system catalog or database
- Contained database/user mapping drift
- Ownership chaining or metadata visibility restriction

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
| where EventID in (7260, 7560, 7860)
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
SELECT HAS_PERMS_BY_NAME(NULL, NULL, 'VIEW SERVER STATE') AS can_view_server_state,
       HAS_PERMS_BY_NAME(NULL, NULL, 'VIEW ANY DEFINITION') AS can_view_any_definition;

SELECT name, state_desc FROM sys.databases ORDER BY name;
SELECT name, type_desc, is_disabled FROM sys.server_principals ORDER BY name;
```

## How to fix

1. Grant the monitoring login minimum required rights: `VIEW SERVER STATE`, `VIEW ANY DEFINITION`, and database CONNECT/VIEW DEFINITION as needed.
2. Remove accidental DENY entries that hide system catalog metadata from the monitoring principal.
3. Fix orphaned contained users or login mappings with `ALTER USER ... WITH LOGIN = ...` where applicable.
4. Rerun the probe and confirm catalog queries succeed without elevating to sysadmin.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to the DBA team if the alert persists after the listed fixes, affects multiple instances/databases, requires downtime, or involves data loss, security key material, cluster failover, or vendor support.

## Related alerts

- [sql-database-securables-config](./sql-database-securables-config.md)
- [sql-error-33111-tde-cert-failure](./sql-error-33111-tde-cert-failure.md)

## References

- KQL: `packs/sql/docs/kql/probe-securables-config.kql`
- Bicep: `packs/sql/alerts/probes.bicep`
- Probe script: `scripts/packs/sql/Public/Test-SqlSecurablesConfig.ps1`
- Threshold defaults: `packs/sql/parameters/thresholds.example.json`
- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Microsoft docs: https://learn.microsoft.com/sql/
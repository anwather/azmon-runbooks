# SQL Server — SQL Server product version is not compliant

**Alert ID:** `sql-engine-service-pack-level`
**Severity:** 3 Warning
**Category:** Engine Service
**Detection mechanism:** DMV probe
**Source SCOM monitor / rule (if any):** `Monitor.DBEngine.ServicePackLevel`

---

## Symptom

The instance product level or update level is below the approved SQL Server baseline.

## What it means

The instance is missing a required service pack, cumulative update, or GDR. This exposes known reliability, security, and AlwaysOn fixes that may already be resolved by patching.

## Common causes

- Instance missed approved CU/GDR patch cycle
- Cluster node patched inconsistently
- Baseline table in pack is newer than local build
- Side-by-side named instance not included in automation

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
| where EventID in (7240, 7540, 7840)
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
SELECT @@SERVERNAME AS server_name,
       SERVERPROPERTY('ProductVersion') AS product_version,
       SERVERPROPERTY('ProductLevel') AS product_level,
       SERVERPROPERTY('ProductUpdateLevel') AS product_update_level,
       SERVERPROPERTY('Edition') AS edition;

SELECT servicename, last_startup_time, service_account, filename
FROM sys.dm_server_services;
```

## How to fix

1. Validate the approved SQL build baseline for this major version and edition.
2. Patch passive cluster/AG replicas first where possible, then fail over and patch former primary.
3. Apply latest approved CU/GDR and reboot if required; success is SERVERPROPERTY values matching baseline.
4. Run a smoke test: SQL connection, Agent jobs enabled, AG synchronized, and application health checks.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to the DBA team if the alert persists after the listed fixes, affects multiple instances/databases, requires downtime, or involves data loss, security key material, cluster failover, or vendor support.

## Related alerts

- [sql-engine-service-status](./sql-engine-service-status.md)
- [sql-engine-status](./sql-engine-status.md)
- [sql-engine-wmi-health](./sql-engine-wmi-health.md)

## References

- KQL: `packs/sql/docs/kql/probe-service-pack-level.kql`
- Bicep: `packs/sql/alerts/probes.bicep`
- Probe script: `scripts/packs/sql/Public/Test-SqlServicePackLevel.ps1`
- Threshold defaults: `packs/sql/parameters/thresholds.example.json`
- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Microsoft docs: https://learn.microsoft.com/sql/
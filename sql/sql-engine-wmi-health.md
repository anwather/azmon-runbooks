# SQL Server — SQL Server WMI health state has errors

**Alert ID:** `sql-engine-wmi-health`
**Severity:** 2 Error
**Category:** Engine Service
**Detection mechanism:** DMV probe
**Source SCOM monitor / rule (if any):** `Monitor.LocalDBEngine.WmiHealth`

---

## Symptom

The SQL Server WMI namespace used for instance discovery and service metadata is not healthy.

## What it means

SQL Server monitoring cannot read the ComputerManagement WMI provider. Discovery, service-name resolution, and per-instance probes may be incomplete even when the engine itself is online.

## Common causes

- SQL Server WMI provider not installed or mismatched version
- root\Microsoft\SqlServer\ComputerManagement namespace corrupt
- WMI repository/service unhealthy
- Monitoring identity lacks local WMI permission
- Firewall/DCOM hardening blocking remote CIM

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
| where EventID in (7210, 7510, 7810)
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
SELECT @@SERVERNAME AS server_name, SYSDATETIME() AS checked_at;
SELECT name, state_desc FROM sys.databases ORDER BY name;
SELECT servicename, status_desc, startup_type_desc FROM sys.dm_server_services;
```

```powershell
Get-CimInstance -Namespace root\Microsoft\SqlServer\ComputerManagement16 -ClassName SqlService |
  Select-Object ServiceName, SQLServiceType, State, StartMode
winmgmt /verifyrepository
```

## How to fix

1. Validate the alert context and affected instance/database.
2. Correct the root cause found in diagnostic output.
3. Rerun the probe or wait for the next sample and confirm the alert clears.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to the DBA team if the alert persists after the listed fixes, affects multiple instances/databases, requires downtime, or involves data loss, security key material, cluster failover, or vendor support.

## Related alerts

- [sql-engine-service-status](./sql-engine-service-status.md)
- [sql-engine-status](./sql-engine-status.md)

## References

- KQL: `packs/sql/docs/kql/probe-wmi-health.kql`
- Bicep: `packs/sql/alerts/probes.bicep`
- Probe script: `scripts/packs/sql/Public/Test-SqlWmiHealth.ps1`
- Threshold defaults: `packs/sql/parameters/thresholds.example.json`
- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Microsoft docs: https://learn.microsoft.com/sql/
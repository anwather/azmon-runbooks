# SQL Server — Database user policy warning

**Alert ID:** `sql-database-user-policy-warning`
**Severity:** 3 Warning
**Category:** Database
**Detection mechanism:** DMV probe
**Source SCOM monitor / rule (if any):** `Monitor.DatabaseWarningUserPolicy.DBWarningUserPolicyState`

---

## Symptom

A custom Policy-Based Management policy for a database evaluated to warning.

## What it means

The customer-defined database warning policy has failed. Treat it as an environment-specific compliance signal and inspect the PBM policy condition and execution history.

## Common causes

- PBM condition changed or no longer matches the approved database baseline
- Database option, owner, permission, or feature state drifted from policy expectations
- Monitoring login cannot evaluate PBM metadata or target database metadata
- Policy execution history contains a failed condition from an earlier state
- The policy is customer-specific and not applicable to this database

## How to diagnose

Start with the 24-hour Log Analytics query to identify the affected computer, instance, database, policy name, and payload. Then connect to the affected SQL Server instance with a DBA account and inspect PBM metadata.

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
SELECT p.name AS policy_name, p.is_enabled, c.name AS condition_name, p.description
FROM msdb.dbo.syspolicy_policies p
LEFT JOIN msdb.dbo.syspolicy_conditions c ON p.condition_id = c.condition_id
ORDER BY p.name;

SELECT TOP (50) h.start_date, h.end_date, h.result, h.exception_message, p.name AS policy_name
FROM msdb.dbo.syspolicy_policy_execution_history h
JOIN msdb.dbo.syspolicy_policies p ON h.policy_id = p.policy_id
ORDER BY h.start_date DESC;

SELECT name, state_desc, is_read_only, owner_sid, compatibility_level
FROM sys.databases
ORDER BY name;
```

## How to fix

1. Confirm the policy is intended to apply to the affected database; disable or retarget only if the database is legitimately out of scope.
2. Read the PBM condition expression and identify the exact database property or facet that failed.
3. Correct the database setting with an explicit `ALTER DATABASE` or security change approved by the DBA owner.
4. If the failure is permissions-related, grant the monitoring login the minimum metadata visibility required to evaluate the policy.
5. Re-evaluate the PBM policy and confirm a new successful row appears in `syspolicy_policy_execution_history`.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to the DBA team and application owner when the failed policy requires a database option change, downtime, security exception, or when the policy intent is unclear. Do not mass-disable PBM policies without owner approval.

## Related alerts

- [sql-database-securables-config](./sql-database-securables-config.md)
- [sql-database-status](./sql-database-status.md)
- [sql-engine-securables-config](./sql-engine-securables-config.md)
- [sql-ag-user-policy-error](./sql-ag-user-policy-error.md)

## References

- KQL: `packs/sql/docs/kql/probe-database-status.kql`
- Bicep: `packs/sql/alerts/probes.bicep`
- Probe script: `scripts/packs/sql/Public/Test-SqlDatabaseStatus.ps1`
- Threshold defaults: `packs/sql/parameters/thresholds.example.json`
- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Microsoft docs: https://learn.microsoft.com/sql/
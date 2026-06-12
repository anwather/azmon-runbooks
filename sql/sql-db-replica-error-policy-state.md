# SQL Server — Database replica user policy error

**Alert ID:** `sql-db-replica-error-policy-state`
**Severity:** 2 Error
**Category:** AlwaysOn
**Detection mechanism:** DMV probe
**Source SCOM monitor / rule (if any):** `Monitor.DatabaseReplicaErrorPolicy.State`

---

## Symptom

A custom Policy-Based Management policy for a database replica evaluated to error.

## What it means

The customer-defined DB replica error policy has failed. The condition is environment-specific; inspect PBM history for the exact rule.

## Common causes

- Custom PBM policy condition changed
- PBM execution history reports a failed condition
- Replica or AG configuration drifted from local standard
- Monitoring login cannot evaluate policy metadata

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
| where EventID in (7150, 7151, 7152, 7450, 7451, 7452, 7750, 7751, 7752)
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| extend Severity = tostring(P.severity), Check = tostring(P.check), Message = tostring(P.message), Payload = P.payload
| extend InstanceName = tostring(P.payload.instanceName), DatabaseName = tostring(P.payload.databaseName)
    // Spans CheckId 150/151/152; narrow to one with: | where toint(P.checkId) == 150
| project TimeGenerated, Computer, InstanceName, DatabaseName, Severity, Check, Message, Payload
| order by TimeGenerated desc
```

```sql
-- T-SQL the DBA runs on the affected instance
SELECT ag.name AS ag_name, ar.replica_server_name, ars.role_desc,
       ars.connected_state_desc, ars.synchronization_health_desc,
       ar.availability_mode_desc, ar.failover_mode_desc
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
LEFT JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id;

SELECT DB_NAME(drs.database_id) AS database_name, drs.synchronization_state_desc,
       drs.synchronization_health_desc, drs.is_suspended, drs.suspend_reason_desc,
       drs.log_send_queue_size, drs.redo_queue_size
FROM sys.dm_hadr_database_replica_states drs
ORDER BY database_name;
```

```powershell
Get-ClusterGroup
Get-ClusterResource | Where-Object {$_.ResourceType -like '*SQL Server Availability Group*'} |
  Get-ClusterParameter | Select-Object ClusterObject, Name, Value
```

## How to fix

1. Confirm the alert is not from planned failover or patching; if planned, verify all replicas return to CONNECTED/SYNCHRONIZED.
2. Restore endpoint and cluster reachability: validate TCP 5022 (or configured endpoint), SQL service state, and WSFC quorum before changing AG state.
3. If data movement is suspended, run `ALTER DATABASE [db] SET HADR RESUME` after correcting the underlying storage/network error; success is `is_suspended = 0` and queues decreasing.
4. For synchronous automatic failover, wait for `synchronization_state_desc = SYNCHRONIZED` and `synchronization_health_desc = HEALTHY` before relying on automatic failover.
5. If primary is lost and quorum is healthy, fail over from a synchronized secondary with `ALTER AVAILABILITY GROUP [ag] FAILOVER`; use forced failover only under DBA incident command.
6. After repair, validate listener routing, SQL Agent jobs on the new primary, and application connection strings.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to DBA, Windows cluster, and network teams if quorum is unstable, lease timeouts recur, synchronous replicas cannot become SYNCHRONIZED, or forced failover/data-loss decisions are being considered.

## Related alerts

- [sql-ag-replicas-connected](./sql-ag-replicas-connected.md)
- [sql-replica-is-connected](./sql-replica-is-connected.md)
- [sql-db-replica-data-sync](./sql-db-replica-data-sync.md)
- [sql-error-35264-ag-data-movement-suspended](./sql-error-35264-ag-data-movement-suspended.md)

## References

- KQL: `packs/sql/docs/kql/probe-database-replica.kql`
- Bicep: `packs/sql/alerts/probes.bicep`
- Probe script: `scripts/packs/sql/Public/Test-SqlDatabaseReplica.ps1`
- Threshold defaults: `packs/sql/parameters/thresholds.example.json`
- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Microsoft docs: https://learn.microsoft.com/sql/
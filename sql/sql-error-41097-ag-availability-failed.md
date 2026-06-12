# SQL Server — SQL error 41097 AG availability failed

**Alert ID:** `sql-error-41097-ag-availability-failed`
**Severity:** 1 Critical
**Category:** AlwaysOn
**Detection mechanism:** Custom Text Logs (ERRORLOG) DCR
**Source SCOM monitor / rule (if any):** `EventRule.AvailabilityGroup.AvailabilityFailed_1_5_Rule`

---

## Symptom

The proposed alert fires when SQL ERRORLOG ingestion sees error 41097. Typical message text: "The availability group failed because a cluster or replica state transition could not complete."

## What it means

AlwaysOn availability, data movement, or WSFC coordination is impaired. Replicas may be stale, failover may be unsafe, and clients may lose connectivity.

## Common causes

- Endpoint connectivity or firewall failure between replicas
- WSFC quorum, node, or resource instability
- Data movement suspended manually or by SQL after error
- Large send/redo queue from storage or network latency
- Replica configured with incompatible failover/availability mode
- Monitoring login cannot read sys.dm_hadr_* state

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
| where ErrorNumber == 41097
| extend InstanceName = tostring(coalesce(InstanceName_s, InstanceName))
| extend DatabaseName = tostring(coalesce(DatabaseName_s, DatabaseName))
| project TimeGenerated, Computer, InstanceName, DatabaseName, ErrorNumber,
          Severity=tostring(coalesce(Severity_s, Severity)), Message=tostring(coalesce(Message, RawData))
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

- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Raw research: `docs/research/sql/raw/agent-c-rules-part1.md` / `agent-c-rules-part2.md` (for rule IDs)
- Microsoft docs: https://learn.microsoft.com/sql/relational-databases/errors-events/mssqlserver-41097-database-engine-error
- Kevin Holman: https://kevinholman.com (search for the SCOM monitor name when relevant)

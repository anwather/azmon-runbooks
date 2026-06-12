# SQL Server — SQL error 9689 Service Broker manager shut down

**Alert ID:** `sql-error-9689-sb-manager-shutdown`
**Severity:** 2 Error
**Category:** Service Broker
**Detection mechanism:** Custom Text Logs (ERRORLOG) DCR
**Source SCOM monitor / rule (if any):** `EventRule.DBEngine.SQL_Server_Service_Broker_Manager_has_shutdown_5_Rule`

---

## Symptom

The proposed alert fires when SQL ERRORLOG ingestion sees error 9689. Typical message text: "The SQL Server Service Broker manager has shut down."

## What it means

Service Broker infrastructure is not running correctly. Queues, activation, Database Mail, Query Notifications, and broker-backed applications can stop processing messages.

## Common causes

- Service Broker disabled in database
- Broker endpoint or transport stopped
- Database master key/certificate/security issue
- Memory pressure prevents broker task startup
- Poison messages or activation procedure failures

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
| where ErrorNumber == 9689
| extend InstanceName = tostring(coalesce(InstanceName_s, InstanceName))
| extend DatabaseName = tostring(coalesce(DatabaseName_s, DatabaseName))
| project TimeGenerated, Computer, InstanceName, DatabaseName, ErrorNumber,
          Severity=tostring(coalesce(Severity_s, Severity)), Message=tostring(coalesce(Message, RawData))
| order by TimeGenerated desc
```

```sql
-- T-SQL the DBA runs on the affected instance
SELECT name, is_broker_enabled, service_broker_guid, state_desc
FROM sys.databases
ORDER BY is_broker_enabled, name;

SELECT name, is_activation_enabled, is_enqueue_enabled, is_receive_enabled
FROM sys.service_queues;

SELECT TOP (50) conversation_handle, transmission_status, enqueue_time
FROM sys.transmission_queue
ORDER BY enqueue_time DESC;
```

## How to fix

1. Confirm broker is enabled only for databases that should use it; do not enable on restored clones without DBA approval.
2. Fix endpoint, certificate, database master key, or route/security errors shown in transmission_status.
3. Resolve poison messages or failed activation procedures and re-enable queues with `ALTER QUEUE ... WITH STATUS = ON`.
4. If broker is disabled unexpectedly, use `ALTER DATABASE [db] SET ENABLE_BROKER WITH ROLLBACK IMMEDIATE` only in an approved outage window.
5. Success is queues enabled, transmission_queue draining, and no new 969x errors.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to the DBA team if the alert persists after the listed fixes, affects multiple instances/databases, requires downtime, or involves data loss, security key material, cluster failover, or vendor support.

## Related alerts

- [sql-error-9694-sb-cannot-start](./sql-error-9694-sb-cannot-start.md)
- [sql-error-9697-sb-cannot-start-on-database](./sql-error-9697-sb-cannot-start-on-database.md)

## References

- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Raw research: `docs/research/sql/raw/agent-c-rules-part1.md` / `agent-c-rules-part2.md` (for rule IDs)
- Microsoft docs: https://learn.microsoft.com/sql/relational-databases/errors-events/mssqlserver-9689-database-engine-error
- Kevin Holman: https://kevinholman.com (search for the SCOM monitor name when relevant)

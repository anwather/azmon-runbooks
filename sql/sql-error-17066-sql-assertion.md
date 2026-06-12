# SQL Server — SQL error 17066 assertion failure

**Alert ID:** `sql-error-17066-sql-assertion`
**Severity:** 2 Error
**Category:** ERRORLOG
**Detection mechanism:** Custom Text Logs (ERRORLOG) DCR
**Source SCOM monitor / rule (if any):** `EventRule.DBEngine.SQL_Server_Assertion_1_5_Rule`

---

## Symptom

The proposed alert fires when SQL ERRORLOG ingestion sees error 17066. Typical message text: "SQL Server Assertion: File, line, expression. SQL Server has encountered an assertion."

## What it means

SQL Server hit an internal assertion. Preserve dumps and error logs; a product defect, corruption, or a repeatable query plan may be involved.

## Common causes

- SQL Server product defect or fixed CU issue
- Database corruption or impossible metadata state
- Problematic query plan/operator hitting assertion path
- Third-party extended procedure/CLR/filter driver
- Hardware/memory corruption

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
| where ErrorNumber == 17066
| extend InstanceName = tostring(coalesce(InstanceName_s, InstanceName))
| extend DatabaseName = tostring(coalesce(DatabaseName_s, DatabaseName))
| project TimeGenerated, Computer, InstanceName, DatabaseName, ErrorNumber,
          Severity=tostring(coalesce(Severity_s, Severity)), Message=tostring(coalesce(Message, RawData))
| order by TimeGenerated desc
```

```sql
-- T-SQL the DBA runs on the affected instance
SELECT TOP (50) ring_buffer_type, timestamp, record
FROM sys.dm_os_ring_buffers
WHERE ring_buffer_type LIKE '%EXCEPTION%' OR ring_buffer_type LIKE '%SCHEDULER%'
ORDER BY timestamp DESC;

SELECT @@VERSION AS sql_version;
```

## How to fix

1. Preserve ERRORLOG, SQL dump files, Windows event logs, and exact query/application timing.
2. Check whether the instance is behind current CU; apply known fixes if the assertion matches a fixed issue.
3. Run CHECKDB on databases referenced near the assertion.
4. If a repeatable query triggers the assertion, capture estimated/actual plan and open vendor/Microsoft support case.
5. Fail over or restart only if the instance is unstable, after collecting dumps and notifying DBA leadership.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to the DBA team if the alert persists after the listed fixes, affects multiple instances/databases, requires downtime, or involves data loss, security key material, cluster failover, or vendor support.

## Related alerts

- [sql-engine-service-status](./sql-engine-service-status.md)
- [sql-engine-status](./sql-engine-status.md)
- [sql-engine-wmi-health](./sql-engine-wmi-health.md)

## References

- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Raw research: `docs/research/sql/raw/agent-c-rules-part1.md` / `agent-c-rules-part2.md` (for rule IDs)
- Microsoft docs: https://learn.microsoft.com/sql/relational-databases/errors-events/mssqlserver-17066-database-engine-error
- Kevin Holman: https://kevinholman.com (search for the SCOM monitor name when relevant)

# SQL Server — Default SQL Server service restarted

**Alert ID:** `sql-engine-service-restart-mssqlserver`
**Severity:** 1 Critical
**Category:** Engine Service
**Detection mechanism:** DCR-Events XPath
**Source SCOM monitor / rule (if any):** `DBEngine.<service-restart> MSSQLSERVER`

---

## Symptom

Service Control Manager or SQL provider indicated the default MSSQLSERVER service restarted.

## What it means

The default instance restarted. In-flight transactions were rolled back, tempdb was recreated, and all connections were dropped.

## Common causes

- Planned patching or cluster failover
- SQL service crash or dump
- Service Control Manager recovery action
- OS reboot or host maintenance
- Startup failure caused repeated recovery attempts

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
| where EventLog == "System" or EventLog == "Application"
| where Source has "MSSQL" or RenderedDescription has "SQL Server"
| where RenderedDescription has_any ("started", "stopped", "terminated")
| project TimeGenerated, Computer, Source, EventID, Level, RenderedDescription
| order by TimeGenerated desc
```

```sql
-- T-SQL the DBA runs on the affected instance
SELECT @@SERVERNAME AS server_name, SYSDATETIME() AS checked_at;
SELECT name, state_desc FROM sys.databases ORDER BY name;
SELECT servicename, status_desc, startup_type_desc FROM sys.dm_server_services;
```

```powershell
Get-Service -Name 'MSSQL*','SQLSERVERAGENT','SQLAgent$*' |
  Select-Object Name, Status, StartType, ServiceName
Get-EventLog -LogName Application -Source 'MSSQLSERVER','SQLSERVERAGENT' -Newest 50 |
  Select-Object TimeGenerated, Source, EventID, EntryType, Message
```

## How to fix

1. Validate the alert context and affected instance/database.
2. Correct the root cause found in diagnostic output.
3. Rerun the probe or wait for the next sample and confirm the alert clears.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to DBA/on-call platform team if the service will not start, system databases are inaccessible, dumps are generated, or restarts recur without a planned change.

## Related alerts

- [sql-engine-service-status](./sql-engine-service-status.md)
- [sql-engine-status](./sql-engine-status.md)
- [sql-engine-wmi-health](./sql-engine-wmi-health.md)

## References

- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Raw research: `docs/research/sql/raw/agent-c-rules-part1.md` / `agent-c-rules-part2.md` (for rule IDs)
- Microsoft docs: https://learn.microsoft.com/sql/relational-databases/system-catalog-views/sys-databases-transact-sql
- Kevin Holman: https://kevinholman.com (search for the SCOM monitor name when relevant)

# SQL Server — SQL Agent alert engine stopped

**Alert ID:** `sql-agent-alert-engine-stopped`
**Severity:** 1 Critical
**Category:** Agent
**Detection mechanism:** DCR-Events XPath
**Source SCOM monitor / rule (if any):** `CollectionRule.Agent.Alert_engine_stopped_due_to_unrecoverable_local_eventlog_errors_1_5_Rule`

---

## Symptom

The SQL Agent alert engine stopped due to unrecoverable local event log errors.

## What it means

Agent may continue running jobs, but alert processing is stopped because local event log access is broken.

## Common causes

- SQL Agent service account cannot connect to SQL
- msdb unavailable or suspect
- Agent service account or Windows rights changed
- Application event log is full, corrupt, or inaccessible
- Recent patch/restart left Agent disabled

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
| where EventLog == "Application"
| where Source has "SQLSERVERAGENT" and EventID == 317

| extend Rendered = tostring(coalesce(RenderedDescription, EventData))
| project TimeGenerated, Computer, Source, EventID, Level, Rendered
| order by TimeGenerated desc
```

```sql
-- T-SQL the DBA runs on the affected instance
SELECT servicename, startup_type_desc, status_desc, last_startup_time, service_account
FROM sys.dm_server_services
WHERE servicename LIKE 'SQL Server Agent%' OR servicename LIKE 'SQL Server (%';

SELECT name, state_desc FROM sys.databases WHERE name = 'msdb';
```

```powershell
Get-Service -Name 'MSSQL*','SQLSERVERAGENT','SQLAgent$*' |
  Select-Object Name, Status, StartType, ServiceName
Get-EventLog -LogName Application -Source 'MSSQLSERVER','SQLSERVERAGENT' -Newest 50 |
  Select-Object TimeGenerated, Source, EventID, EntryType, Message
```

## How to fix

1. Verify the Database Engine and msdb are online before restarting Agent.
2. Fix service account logon rights, password, SPN, or SQL login mapping if Agent cannot authenticate.
3. Start Agent with `Start-Service SQLSERVERAGENT` or the named-instance `SQLAgent$Instance`; success is service Status=Running.
4. If event log errors persist, repair Application log access/size/ACLs and restart Agent to reinitialize alert processing.
5. Confirm critical jobs have resumed and no enabled jobs were missed during outage.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to DBA/on-call platform team if the service will not start, system databases are inaccessible, dumps are generated, or restarts recur without a planned change.

## Related alerts

- [sql-agent-service-status](./sql-agent-service-status.md)
- [sql-agent-unable-to-connect](./sql-agent-unable-to-connect.md)
- [sql-agent-job-failed](./sql-agent-job-failed.md)
- [sql-agent-job-duration](./sql-agent-job-duration.md)

## References

- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Raw research: `docs/research/sql/raw/agent-c-rules-part1.md` / `agent-c-rules-part2.md` (for rule IDs)
- Microsoft docs: https://learn.microsoft.com/sql/relational-databases/system-tables/dbo-sysjobs-transact-sql
- Microsoft docs: https://learn.microsoft.com/sql/relational-databases/system-tables/dbo-sysjobhistory-transact-sql
- Kevin Holman: https://kevinholman.com (search for the SCOM monitor name when relevant)

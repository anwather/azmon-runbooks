# SQL Server — SQL Server Agent service stopped

**Alert ID:** `sql-agent-service-status`
**Severity:** 2 Error
**Category:** Agent
**Detection mechanism:** DMV probe
**Source SCOM monitor / rule (if any):** `Monitor.Agent.ServiceStatus`

---

## Symptom

SQL Server Agent is stopped or its state cannot be detected.

## What it means

Scheduled jobs, alerts, operators, replication agents, log shipping jobs, and maintenance plans may not run until Agent is running. Express editions do not include Agent and should be excluded.

## Common causes

- SQL Agent service stopped or disabled
- Agent service account cannot log on
- Agent cannot connect to Database Engine
- msdb unavailable or in recovery
- Express edition where Agent is unsupported

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
| where EventID in (7200, 7500, 7800)
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

- [sql-agent-unable-to-connect](./sql-agent-unable-to-connect.md)
- [sql-agent-job-failed](./sql-agent-job-failed.md)
- [sql-agent-job-duration](./sql-agent-job-duration.md)

## References

- KQL: `packs/sql/docs/kql/probe-service-state.kql`
- Bicep: `packs/sql/alerts/probes.bicep`
- Probe script: `scripts/packs/sql/Public/Test-SqlServiceState.ps1`
- Threshold defaults: `packs/sql/parameters/thresholds.example.json`
- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Microsoft docs: https://learn.microsoft.com/sql/
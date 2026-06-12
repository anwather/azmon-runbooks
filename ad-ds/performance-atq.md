# AD DS — ATQ thread pool usage

**Alert ID:** `ad-ds-performance-atq`
**Severity:** 3 (Warning)
**Category:** Performance
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Performance.Atq.AvgThreads.Monitor`

---

## Symptom

The alert description is: "ATQ thread pool usage is currently over threshold." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

ATQ thread pool usage is high. LDAP, Kerberos, and replication operations can queue or time out when LSASS worker threads are saturated.

## Common causes

- High LDAP bind/search concurrency
- Expensive queries
- Authentication storm
- Replication backlog
- CPU starvation

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1060, 1360, 1660)
| extend CheckId = case(EventID in (1060, 1360, 1660), 60, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
Get-Counter "\NTDS\ATQ Threads LDAP"
Get-Counter "\NTDS\ATQ Outstanding Queued Requests"
Get-Counter "\Process(lsass)\% Processor Time"
dcdiag /test:Services /v
```

## How to fix

1. Redirect heavy LDAP clients
2. tune expensive filters/paging/indexes
3. add DC capacity
4. collect diagnostics before rebooting LSASS.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [performance-lsass-cpu](./performance-lsass-cpu.md)
- [performance-dns-process-cpu](./performance-dns-process-cpu.md)

## References

- KQL: `packs/ad-ds/docs/kql/performance-atq.kql`
- Bicep: `packs/ad-ds/alerts/performance.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-AtqThreads.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/component-updates/performance-tuning-active-directory
- Microsoft docs: https://learn.microsoft.com/troubleshoot/windows-server/identity/high-lsassexe-cpu-utilization-troubleshooting
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

# AD DS — Time service events

**Alert ID:** `ad-ds-event-collection-time`
**Severity:** 3 (Warning)
**Category:** Event collection
**Detection mechanism:** Event-log XPath collection
**Source SCOM monitor (if any):** `SCOM event collection rules`

---

## Symptom

The alert description is: "Windows Time service synchronization events were detected." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

Windows Time service synchronization events were logged. They often precede Kerberos and secure-channel failures.

## Common causes

- W32Time cannot reach source
- PDC source changed
- NTP blocked
- Large correction
- Domain hierarchy broken

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where EventLog == "System"
| where Source in~ ("Microsoft-Windows-Time-Service", "Time-Service")
| where EventID in (21, 34, 36)
| project TimeGenerated, Computer, EventLog, EventID, Source, RenderedDescription
| order by TimeGenerated desc
```

```powershell
wevtutil qe System /q:"*[System[(EventID=21 or EventID=34 or EventID=36)]]" /c:10 /f:text
w32tm /query /status
w32tm /query /source
w32tm /monitor
```

## How to fix

1. Restore correct source
2. fix UDP 123 reachability
3. return member DCs to domhier
4. success is stable offset and no new events.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [event-collection-kdc](./event-collection-kdc.md)
- [event-collection-netlogon-dns](./event-collection-netlogon-dns.md)
- [event-collection-rid-pool](./event-collection-rid-pool.md)
- [event-collection-gpo-failures](./event-collection-gpo-failures.md)
- [event-collection-dnsapi](./event-collection-dnsapi.md)

## References

- KQL: `packs/ad-ds/docs/kql/event-collection-time.kql`
- Bicep: `packs/ad-ds/alerts/eventCollection.bicep`
- Probe script (if applicable): none; this alert uses collected Windows events or native perf counters.
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/wevtutil
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/dcdiag
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

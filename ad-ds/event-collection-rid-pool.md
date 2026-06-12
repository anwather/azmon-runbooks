# AD DS — RID pool events

**Alert ID:** `ad-ds-event-collection-rid-pool`
**Severity:** 1 (Critical)
**Category:** Event collection
**Detection mechanism:** Event-log XPath collection
**Source SCOM monitor (if any):** `SCOM event collection rules`

---

## Symptom

The alert description is: "SAM RID pool events indicating account creation risk were detected." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

Directory-Services-SAM logged RID pool events indicating account creation risk. Treat this as domain-level.

## Common causes

- RID Master unreachable
- RID allocation failed
- Domain RID ceiling approaching
- RID Manager replication inconsistency
- Bulk object creation depleted pools

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where EventLog == "System"
| where Source in~ ("Directory-Services-SAM", "Microsoft-Windows-Directory-Services-SAM")
| where EventID in (16641, 16642, 16643, 16651)
| project TimeGenerated, Computer, EventLog, EventID, Source, RenderedDescription
| order by TimeGenerated desc
```

```powershell
wevtutil qe System /q:"*[System[(EventID=16641 or EventID=16642 or EventID=16643 or EventID=16651)]]" /c:10 /f:text
netdom query fsmo
dcdiag /test:RidManager /v
repadmin /replsummary
```

## How to fix

1. Stabilize RID Master
2. pause automation creating accounts
3. resolve RID Manager errors
4. escalate domain-wide RID exhaustion indicators immediately.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [event-collection-kdc](./event-collection-kdc.md)
- [event-collection-netlogon-dns](./event-collection-netlogon-dns.md)
- [event-collection-gpo-failures](./event-collection-gpo-failures.md)
- [event-collection-time](./event-collection-time.md)
- [event-collection-dnsapi](./event-collection-dnsapi.md)

## References

- KQL: `packs/ad-ds/docs/kql/event-collection-rid-pool.kql`
- Bicep: `packs/ad-ds/alerts/eventCollection.bicep`
- Probe script (if applicable): none; this alert uses collected Windows events or native perf counters.
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/wevtutil
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/dcdiag
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

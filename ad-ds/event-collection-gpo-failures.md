# AD DS — Group Policy failure events

**Alert ID:** `ad-ds-event-collection-gpo-failures`
**Severity:** 2 (Error)
**Category:** Event collection
**Detection mechanism:** Event-log XPath collection
**Source SCOM monitor (if any):** `SCOM event collection rules`

---

## Symptom

The alert description is: "Authentication-impacting Group Policy failure events were detected." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

GroupPolicy failure events were logged. Users or computers may not receive security settings, scripts, or administrative templates.

## Common causes

- SYSVOL/Netlogon unavailable
- DNS cannot locate DC
- GPO permissions or missing GPT.ini
- SMB/network blocked
- Client-side extension failure

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where EventLog == "System"
| where Source in~ ("Microsoft-Windows-GroupPolicy", "GroupPolicy")
| where EventID in (1030, 1054, 1097, 1109)
| project TimeGenerated, Computer, EventLog, EventID, Source, RenderedDescription
| order by TimeGenerated desc
```

```powershell
wevtutil qe System /q:"*[System[(EventID=1030 or EventID=1054 or EventID=1097 or EventID=1109)]]" /c:10 /f:text
gpupdate /force
gpresult /h gpresult.html
net view \$env:USERDNSDOMAIN\SYSVOL
```

## How to fix

1. Use event text to find failing GPO/path
2. restore SYSVOL/DNS/SMB
3. fix GPO ACL or GPT content
4. success is gpupdate clean.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [event-collection-kdc](./event-collection-kdc.md)
- [event-collection-netlogon-dns](./event-collection-netlogon-dns.md)
- [event-collection-rid-pool](./event-collection-rid-pool.md)
- [event-collection-time](./event-collection-time.md)
- [event-collection-dnsapi](./event-collection-dnsapi.md)

## References

- KQL: `packs/ad-ds/docs/kql/event-collection-gpo-failures.kql`
- Bicep: `packs/ad-ds/alerts/eventCollection.bicep`
- Probe script (if applicable): none; this alert uses collected Windows events or native perf counters.
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/wevtutil
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/dcdiag
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

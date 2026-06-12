# AD DS — KDC critical events

**Alert ID:** `ad-ds-event-collection-kdc`
**Severity:** 1 (Critical)
**Category:** Event collection
**Detection mechanism:** Event-log XPath collection
**Source SCOM monitor (if any):** `SCOM event collection rules`

---

## Symptom

The alert description is: "Kerberos KDC critical events were detected." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

KDC critical events were logged. Kerberos authentication can fail for affected accounts, services, trusts, or encryption policies.

## Common causes

- Duplicate/missing SPN
- Unsupported encryption or key mismatch
- Trust/krbtgt password problem
- Clock skew
- Misconfigured service account

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where EventLog == "System"
| where Source in~ ("Microsoft-Windows-Kerberos-Key-Distribution-Center")
| where EventID in (5, 6, 7, 10)
| project TimeGenerated, Computer, EventLog, EventID, Source, RenderedDescription
| order by TimeGenerated desc
```

```powershell
wevtutil qe System /q:"*[System[Provider[@Name='Microsoft-Windows-Kerberos-Key-Distribution-Center'] and (EventID=5 or EventID=6 or EventID=7 or EventID=10)]]" /c:10 /f:text
dcdiag /test:Advertising /v
setspn -X
nltest /sc_query:$env:USERDNSDOMAIN
```

## How to fix

1. Use event text to identify account/SPN/domain
2. fix duplicate SPNs
3. correct encryption/key issues
4. validate trust from both sides before resets.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [event-collection-netlogon-dns](./event-collection-netlogon-dns.md)
- [event-collection-rid-pool](./event-collection-rid-pool.md)
- [event-collection-gpo-failures](./event-collection-gpo-failures.md)
- [event-collection-time](./event-collection-time.md)
- [event-collection-dnsapi](./event-collection-dnsapi.md)

## References

- KQL: `packs/ad-ds/docs/kql/event-collection-kdc.kql`
- Bicep: `packs/ad-ds/alerts/eventCollection.bicep`
- Probe script (if applicable): none; this alert uses collected Windows events or native perf counters.
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/wevtutil
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/dcdiag
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

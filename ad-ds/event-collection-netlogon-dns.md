# AD DS — Netlogon DNS registration events

**Alert ID:** `ad-ds-event-collection-netlogon-dns`
**Severity:** 2 (Error)
**Category:** Event collection
**Detection mechanism:** Event-log XPath collection
**Source SCOM monitor (if any):** `SCOM event collection rules`

---

## Symptom

The alert description is: "Netlogon DNS registration or site-location events were detected." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

Netlogon DNS registration or site-location events were logged. DC locator records may be missing or stale.

## Common causes

- Dynamic updates disabled/denied
- Netlogon cannot reach DNS
- Zone permissions wrong
- Stale site/SRV records
- DNS service issue

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where EventLog == "System"
| where Source in~ ("NETLOGON", "Netlogon")
| where EventID in (5773, 5784, 5786)
| project TimeGenerated, Computer, EventLog, EventID, Source, RenderedDescription
| order by TimeGenerated desc
```

```powershell
wevtutil qe System /q:"*[System[(EventID=5773 or EventID=5784 or EventID=5786)]]" /c:10 /f:text
nltest /dsregdns
dcdiag /test:DNS /v
Resolve-DnsName _ldap._tcp.dc._msdcs.$env:USERDNSDOMAIN
```

## How to fix

1. Fix DNS reachability/permissions
2. run nltest /dsregdns
3. remove stale retired-DC records
4. success is clean dcdiag DNS and no new events.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [event-collection-kdc](./event-collection-kdc.md)
- [event-collection-rid-pool](./event-collection-rid-pool.md)
- [event-collection-gpo-failures](./event-collection-gpo-failures.md)
- [event-collection-time](./event-collection-time.md)
- [event-collection-dnsapi](./event-collection-dnsapi.md)

## References

- KQL: `packs/ad-ds/docs/kql/event-collection-netlogon-dns.kql`
- Bicep: `packs/ad-ds/alerts/eventCollection.bicep`
- Probe script (if applicable): none; this alert uses collected Windows events or native perf counters.
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/wevtutil
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/dcdiag
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

# AD DS — DNSAPI dynamic update events

**Alert ID:** `ad-ds-event-collection-dnsapi`
**Severity:** 2 (Error)
**Category:** Event collection
**Detection mechanism:** Event-log XPath collection
**Source SCOM monitor (if any):** `SCOM event collection rules`

---

## Symptom

The alert description is: "DNS dynamic update failure events were detected." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

DNSAPI dynamic update failures were logged. The DC or clients may be unable to register required DNS records.

## Common causes

- Dynamic update refused by ACL
- DNS server unreachable
- Record conflict/stale owner
- Secure update/Kerberos failure
- Client using non-AD DNS

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where EventLog == "System"
| where Source in~ ("DNSAPI", "Dnsapi")
| where EventID in (11150, 11151, 11152, 11153, 11164, 11165)
| project TimeGenerated, Computer, EventLog, EventID, Source, RenderedDescription
| order by TimeGenerated desc
```

```powershell
wevtutil qe System /q:"*[System[(EventID=11150 or EventID=11151 or EventID=11152 or EventID=11153 or EventID=11164 or EventID=11165)]]" /c:10 /f:text
ipconfig /registerdns
nltest /dsregdns
dcdiag /test:DNS /v
```

## How to fix

1. Correct NIC DNS and zone update permissions
2. remove stale/conflicting records
3. re-register DNS
4. success is no new DNSAPI failures and valid records.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [event-collection-kdc](./event-collection-kdc.md)
- [event-collection-netlogon-dns](./event-collection-netlogon-dns.md)
- [event-collection-rid-pool](./event-collection-rid-pool.md)
- [event-collection-gpo-failures](./event-collection-gpo-failures.md)
- [event-collection-time](./event-collection-time.md)

## References

- KQL: `packs/ad-ds/docs/kql/event-collection-dnsapi.kql`
- Bicep: `packs/ad-ds/alerts/eventCollection.bicep`
- Probe script (if applicable): none; this alert uses collected Windows events or native perf counters.
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/wevtutil
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/dcdiag
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

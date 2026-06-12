# AD DS — Network adapter DNS

**Alert ID:** `ad-ds-dns-adapters`
**Severity:** 2 (Error)
**Category:** DNS
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Configuration.NetworkAdapters.DNS.Monitor`

---

## Symptom

The alert description is: "Configured DNS servers are not sufficiently reachable." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

Too few reachable DNS servers are configured on DC network adapters. Bad NIC DNS breaks DC locator and replication name resolution.

## Common causes

- NIC points to public DNS
- Preferred DNS offline
- IPv6 DNS misconfigured
- Stale adapter
- DNS Client service stopped

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1090, 1390, 1690)
| extend CheckId = case(EventID in (1090, 1390, 1690), 90, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4,IPv6
Resolve-DnsName _ldap._tcp.dc._msdcs.$env:USERDNSDOMAIN
dcdiag /test:DNS /v
nltest /dsgetdc:$env:USERDNSDOMAIN /force
```

## How to fix

1. Configure NIC DNS to healthy AD DNS servers
2. remove public/unreachable DNS
3. validate SRV lookups
4. run nltest /dsregdns if records need registration.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- None.

## References

- KQL: `packs/ad-ds/docs/kql/dns-adapters.kql`
- Bicep: `packs/ad-ds/alerts/dns.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-NetworkAdapterDns.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/networking/dns/troubleshoot/troubleshoot-dns-server
- Microsoft docs: https://learn.microsoft.com/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

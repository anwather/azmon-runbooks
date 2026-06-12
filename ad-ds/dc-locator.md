# AD DS — DC Locator

**Alert ID:** `ad-ds-dc-locator`
**Severity:** 1 (Critical)
**Category:** SYSVOL/DC Locator
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Availability.DCLocator.ServiceCheck`

---

## Symptom

The alert description is: "DC Locator is currently returning an unreachable DC." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

DC Locator returned an unreachable DC. Clients rely on this for logon, LDAP, Kerberos, and site-aware service discovery.

## Common causes

- Missing/stale DNS SRV records
- Netlogon not registering
- Bad site/subnet mapping
- Firewall blocks selected DC
- Demoted DC records remain

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1121, 1421, 1721)
| extend CheckId = case(EventID in (1121, 1421, 1721), 121, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
nltest /dsgetdc:$env:USERDNSDOMAIN /force
nltest /dclist:$env:USERDNSDOMAIN
dcdiag /test:DNS /v
nltest /dsregdns
```

## How to fix

1. Remove stale DC records
2. restart Netlogon or run nltest /dsregdns
3. correct sites/subnets
4. success is nltest returning a reachable site-appropriate DC.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [sysvol-share](./sysvol-share.md)
- [dfsr-wmi](./dfsr-wmi.md)

## References

- KQL: `packs/ad-ds/docs/kql/dc-locator.kql`
- Bicep: `packs/ad-ds/alerts/sysvol.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-DcLocator.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/troubleshoot/troubleshoot-missing-sysvol-and-netlogon-shares
- Microsoft docs: https://learn.microsoft.com/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

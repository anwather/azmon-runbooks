# AD DS — LDAP bind availability

**Alert ID:** `ad-ds-ldap-bind-availability`
**Severity:** 1 (Critical)
**Category:** LDAP/GC
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Availability.Bind.Monitor`

---

## Symptom

The alert description is: "Local LDAP bind is currently failing." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

The local DC cannot complete an LDAP RootDSE bind. Clients and applications using this DC may fail directory lookups and authentication flows.

## Common causes

- NTDS stopped or LSASS unresponsive
- Local DNS problem
- LDAP blocked locally
- DC not advertising
- Kerberos/secure channel failure

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1050, 1350, 1650)
| extend CheckId = case(EventID in (1050, 1350, 1650), 50, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
Get-Service NTDS,Netlogon,KDC
Get-ADRootDSE -Server localhost
dcdiag /test:Services /test:Advertising /v
nltest /dsgetdc:$env:USERDNSDOMAIN /server:$env:COMPUTERNAME
```

## How to fix

1. Start required services
2. fix local DNS and advertising failures
3. collect diagnostics before rebooting hung LSASS
4. success is Get-ADRootDSE returning RootDSE.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [ldap-bind-time](./ldap-bind-time.md)
- [gc-search-availability](./gc-search-availability.md)
- [gc-search-performance](./gc-search-performance.md)

## References

- KQL: `packs/ad-ds/docs/kql/ldap-bind.kql`
- Bicep: `packs/ad-ds/alerts/ldap-gc.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-LdapBind.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/plan/global-catalog
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/component-updates/performance-tuning-active-directory
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

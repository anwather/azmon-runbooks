# AD DS — Global Catalog search availability

**Alert ID:** `ad-ds-gc-search-availability`
**Severity:** 1 (Critical)
**Category:** LDAP/GC
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Availability.GCResponse.Monitor`

---

## Symptom

The alert description is: "Global Catalog search is currently failing." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

Global Catalog search is failing on a DC expected to be a GC. Universal group expansion and cross-domain searches can fail.

## Common causes

- DC is not a ready GC
- NTDS/Netlogon not advertising GC
- Port 3268 blocked
- Replication not converged
- Stale GC SRV records

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1052, 1352, 1652)
| extend CheckId = case(EventID in (1052, 1352, 1652), 52, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
nltest /dsgetdc:$env:USERDNSDOMAIN /gc /force
Get-ADRootDSE -Server localhost | Select isGlobalCatalogReady
dcdiag /test:Advertising /v
netstat -ano | findstr ":3268"
```

## How to fix

1. Confirm GC role is intended and ready
2. repair replication/advertising
3. fix listener/firewall problems
4. update monitoring if GC was removed intentionally.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [ldap-bind-availability](./ldap-bind-availability.md)
- [ldap-bind-time](./ldap-bind-time.md)
- [gc-search-performance](./gc-search-performance.md)

## References

- KQL: `packs/ad-ds/docs/kql/gc-search.kql`
- Bicep: `packs/ad-ds/alerts/ldap-gc.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-GcSearchAvailability.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/plan/global-catalog
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/component-updates/performance-tuning-active-directory
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

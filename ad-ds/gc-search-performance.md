# AD DS — Global Catalog search performance

**Alert ID:** `ad-ds-gc-search-performance`
**Severity:** 3 (Warning)
**Category:** LDAP/GC
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Performance.GCResponse.Monitor`

---

## Symptom

The alert description is: "Global Catalog search time is currently over threshold." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

Global Catalog search is slow. Users can see delayed logons, address-book lookup, and cross-domain authorization.

## Common causes

- LSASS CPU/memory pressure
- Expensive LDAP searches
- Replication backlog
- WAN latency
- Insufficient GC capacity

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1053, 1353, 1653)
| extend CheckId = case(EventID in (1053, 1353, 1653), 53, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
Measure-Command { Get-ADRootDSE -Server localhost }
Get-Counter "\Process(lsass)\% Processor Time"
repadmin /replsummary
dcdiag /test:Advertising /v
```

## How to fix

1. Validate latency from DC and client sites
2. fix replication/advertising
3. tune heavy LDAP clients
4. add GC capacity if sustained.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [ldap-bind-availability](./ldap-bind-availability.md)
- [ldap-bind-time](./ldap-bind-time.md)
- [gc-search-availability](./gc-search-availability.md)

## References

- KQL: `packs/ad-ds/docs/kql/gc-search.kql`
- Bicep: `packs/ad-ds/alerts/ldap-gc.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-GcSearchPerformance.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/plan/global-catalog
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/component-updates/performance-tuning-active-directory
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

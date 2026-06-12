# AD DS — RID pool exhaustion approaching

**Alert ID:** `ad-ds-rid-pool`
**Severity:** 2 (Error)
**Category:** RID Pool
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `SCOM RID Pool % Free perf collection (no UnitMonitor)`

---

## Symptom

The alert description is: "RID pool free percentage on the RID Master is below the warning/error threshold. Only the RID Master role-holder emits meaningful data; other DCs are skipped. Probe checkId 130." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

The RID Master reports low RID pool percentage. If RID allocation fails, DCs cannot create new users, groups, computers, or other security principals.

## Common causes

- RID Master unreachable
- Bulk account creation spike
- RID block allocation failures
- RID Manager/FSMO inconsistency
- Domain nearing RID ceiling

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1130, 1430, 1730)
| extend CheckId = case(EventID in (1130, 1430, 1730), 130, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
netdom query fsmo
dcdiag /test:RidManager /v
repadmin /showrepl
Get-ADDomain | Select RIDMaster
```

## How to fix

1. Stabilize RID Master and replication
2. pause bulk account creation
3. resolve dcdiag RID Manager errors
4. escalate immediately for suspected domain RID exhaustion.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- None.

## References

- KQL: `packs/ad-ds/docs/kql/rid-pool.kql`
- Bicep: `packs/ad-ds/alerts/ridPool.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-RidPoolFree.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/managing-rid-issuance
- Microsoft docs: https://learn.microsoft.com/troubleshoot/windows-server/active-directory/fsmo-roles
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

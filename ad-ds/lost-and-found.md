# AD DS — LostAndFound object count

**Alert ID:** `ad-ds-lost-and-found`
**Severity:** 3 (Warning)
**Category:** Lost & Found
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `LostObjectCount.Monitor`

---

## Symptom

The alert description is: "LostAndFound object count is currently over threshold." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

CN=LostAndFound has too many objects. This indicates orphaned objects from missing parents, restore, or replication conflict.

## Common causes

- Deleted parent before child replicated
- Lingering replication after restore
- Application wrote under missing container
- Manual recovery left orphans
- Bulk-move conflict

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1110, 1410, 1710)
| extend CheckId = case(EventID in (1110, 1410, 1710), 110, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
repadmin /replsummary
dcdiag /test:Replications /v
dsquery * "CN=LostAndFound,DC=example,DC=com" -scope onelevel
repadmin /showobjmeta $env:COMPUTERNAME "CN=LostAndFound,DC=example,DC=com"
```

## How to fix

1. Review each object owner/type
2. move valid objects to correct parent
3. delete obsolete orphans after approval
4. fix replication so new objects stop appearing.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- None.

## References

- KQL: `packs/ad-ds/docs/kql/lost-and-found.kql`
- Bicep: `packs/ad-ds/alerts/lostFound.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-LostAndFoundCount.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/troubleshoot/troubleshooting-active-directory-replication-problems
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

# AD DS — FSMO role consistency

**Alert ID:** `ad-ds-fsmo-consistency`
**Severity:** 3 (Warning)
**Category:** FSMO
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Configuration.FsmoConsistency.*.Monitor`

---

## Symptom

The alert description is: "FSMO ownership is inconsistent with the PDC view." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

This DC sees FSMO ownership differently than the PDC emulator. Inconsistent metadata can break RID allocation and schema/domain operations.

## Common causes

- Replication lag
- Recent role transfer/seizure not replicated
- Stale retired holder metadata
- PDC emulator unhealthy

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1030, 1330, 1630, 1031, 1331, 1631, 1032, 1332, 1632, 1033, 1333, 1633)
| extend CheckId = case(EventID in (1030, 1330, 1630), 30, EventID in (1031, 1331, 1631), 31, EventID in (1032, 1332, 1632), 32, EventID in (1033, 1333, 1633), 33, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
netdom query fsmo
Get-ADDomain | Select PDCEmulator,RIDMaster,InfrastructureMaster
Get-ADForest | Select SchemaMaster,DomainNamingMaster
repadmin /showrepl
repadmin /syncall /AdeP
```

## How to fix

1. Repair replication first
2. verify role transfer completed
3. clean stale server metadata only after validation
4. do not seize roles unless owner is unrecoverable.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [fsmo-ping-availability](./fsmo-ping-availability.md)
- [fsmo-bind-availability](./fsmo-bind-availability.md)
- [fsmo-ping-performance](./fsmo-ping-performance.md)
- [fsmo-bind-performance](./fsmo-bind-performance.md)

## References

- KQL: `packs/ad-ds/docs/kql/fsmo-consistency.kql`
- Bicep: `packs/ad-ds/alerts/fsmo.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-FsmoConsistency.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/troubleshoot/windows-server/active-directory/fsmo-roles
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/netdom
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

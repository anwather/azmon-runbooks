# AD DS — NTDS.dit disk free space

**Alert ID:** `ad-ds-database-ntds-dit-disk`
**Severity:** 2 (Error)
**Category:** Database
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Availability.DiskSpace.DIT.Monitor`

---

## Symptom

The alert description is: "The NTDS.dit drive is currently below free-space threshold." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

The NTDS.dit volume is below threshold. If it fills, AD DS can stop accepting writes or fail to start.

## Common causes

- Undersized volume
- Shadow copies/backups/dumps consuming space
- Unexpected database growth
- Antivirus or installer cache
- Old maintenance files

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1040, 1340, 1640)
| extend CheckId = case(EventID in (1040, 1340, 1640), 40, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters | Select "DSA Database file"
Get-Volume
Get-ChildItem -Force C:\
dcdiag /test:Services /v
```

## How to fix

1. Remove non-AD files and move backups/dumps
2. extend the volume if needed
3. plan database relocation only with AD engineering
4. success is free space above threshold.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [database-edb-log-disk](./database-edb-log-disk.md)

## References

- KQL: `packs/ad-ds/docs/kql/database-disk.kql`
- Bicep: `packs/ad-ds/alerts/database.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-AdDatabaseDiskSpace.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/ad-forest-recovery-backup-system-state
- Microsoft docs: https://learn.microsoft.com/windows-server/storage/disk-management/overview-of-disk-management
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

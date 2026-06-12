# AD DS — EDB log disk free space

**Alert ID:** `ad-ds-database-edb-log-disk`
**Severity:** 2 (Error)
**Category:** Database
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Availability.DiskSpace.Log.Monitor`

---

## Symptom

The alert description is: "The EDB log drive is currently below free-space threshold." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

The AD log volume is below threshold. Full log disks can stall transactions and stop NTDS.

## Common causes

- Backups failing so logs are retained
- Logs on small system volume
- Bulk changes producing logs
- Backup/AV locking files
- Unexpected files on log volume

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1041, 1341, 1641)
| extend CheckId = case(EventID in (1041, 1341, 1641), 41, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters | Select "Database log files path"
Get-Volume
wbadmin get versions
dcdiag /test:Services /v
```

## How to fix

1. Repair system-state backups so logs truncate
2. do not manually delete active EDB logs
3. extend the volume
4. if NTDS stopped, preserve logs and escalate.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [database-ntds-dit-disk](./database-ntds-dit-disk.md)

## References

- KQL: `packs/ad-ds/docs/kql/database-disk.kql`
- Bicep: `packs/ad-ds/alerts/database.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-AdLogDiskSpace.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/ad-forest-recovery-backup-system-state
- Microsoft docs: https://learn.microsoft.com/windows-server/storage/disk-management/overview-of-disk-management
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

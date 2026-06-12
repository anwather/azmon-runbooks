# AD DS — SYSVOL share availability

**Alert ID:** `ad-ds-sysvol-share`
**Severity:** 1 (Critical)
**Category:** SYSVOL/DC Locator
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Availability.SysVol.ServiceCheck`

---

## Symptom

The alert description is: "The SYSVOL share is currently unavailable." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

SYSVOL is unavailable. Clients and DCs need it for GPO templates/scripts and Netlogon scripts.

## Common causes

- DFSR/FRS not initialized
- SYSVOL folder missing or ACL changed
- Netlogon not sharing
- Disk/AV blocking path
- Recent promotion initial sync incomplete

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1120, 1420, 1720)
| extend CheckId = case(EventID in (1120, 1420, 1720), 120, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
net share
net view \$env:COMPUTERNAME
dcdiag /test:SysVolCheck /test:Advertising /v
dfsrdiag ReplicationState
```

## How to fix

1. Restore SYSVOL/NETLOGON shares
2. repair DFSR/FRS backlog or initial sync
3. restart Netlogon after content exists
4. use authoritative SYSVOL restore only with AD engineering.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [dc-locator](./dc-locator.md)
- [dfsr-wmi](./dfsr-wmi.md)

## References

- KQL: `packs/ad-ds/docs/kql/sysvol-share.kql`
- Bicep: `packs/ad-ds/alerts/sysvol.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-SysvolShare.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/troubleshoot/troubleshoot-missing-sysvol-and-netlogon-shares
- Microsoft docs: https://learn.microsoft.com/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

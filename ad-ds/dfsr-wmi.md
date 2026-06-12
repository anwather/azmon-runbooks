# AD DS — DFSR WMI provider

**Alert ID:** `ad-ds-dfsr-wmi`
**Severity:** 2 (Error)
**Category:** SYSVOL/DC Locator
**Detection mechanism:** Event-log XPath collection
**Source SCOM monitor (if any):** `AvailabilityEssentialService.DFSR.WMICheck`

---

## Symptom

The alert description is: "DFSR WMI provider failure event is currently uncleared." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

DFSR Event 6104 was seen without a later 6102 clear. DFSR WMI provider registration is broken, affecting DFSR management/monitoring visibility.

## Common causes

- DFSR WMI provider registration failed
- WMI repository/provider issue
- DFSR service unhealthy
- MOF registration problem
- Recent DFSR role change

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where EventLog == "DFS Replication" and EventID in (6102, 6104)
| summarize arg_max(TimeGenerated, *) by Computer
| where EventID == 6104
| project TimeGenerated, Computer, EventLog, EventID, Source, RenderedDescription
```

```powershell
wevtutil qe "DFS Replication" /q:"*[System[(EventID=6104 or EventID=6102)]]" /c:20 /f:text
Get-Service DFSR
dfsrdiag PollAD
wmic /namespace:\\root\microsoftdfs path DfsrReplicatedFolderInfo get ReplicatedFolderName,State
```

## How to fix

1. Restart DFSR and recheck
2. re-register DFSR WMI provider/MOF using Microsoft guidance
3. repair WMI only after provider repair fails
4. success is Event 6102 and WMI query success.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [sysvol-share](./sysvol-share.md)
- [dc-locator](./dc-locator.md)

## References

- KQL: `packs/ad-ds/docs/kql/dfsr-wmi.kql`
- Bicep: `packs/ad-ds/alerts/sysvol.bicep`
- Probe script (if applicable): none; this alert uses collected Windows events or native perf counters.
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/troubleshoot/troubleshoot-missing-sysvol-and-netlogon-shares
- Microsoft docs: https://learn.microsoft.com/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

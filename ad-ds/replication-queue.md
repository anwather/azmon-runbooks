# AD DS — Replication queue depth

**Alert ID:** `ad-ds-replication-queue`
**Severity:** 2 (Error)
**Category:** Replication
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Performance.Replication.Queue.Monitor`

---

## Symptom

The alert description is: "AD replication queue depth currently exceeds threshold." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

The DC is accumulating pending replication operations faster than it drains them. Users may see delayed password, group, DNS, or GPO changes.

## Common causes

- Slow or unreachable replication partner
- Large burst of directory changes
- LSASS CPU or disk bottleneck
- Restrictive site-link schedule
- RPC/network packet loss

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1002, 1302, 1602)
| extend CheckId = case(EventID in (1002, 1302, 1602), 2, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
repadmin /queue
repadmin /replsummary
Get-Counter "\NTDS\DRA Pending Replication Operations"
repadmin /showrepl
```

## How to fix

1. Identify the partner with the backlog
2. fix network/RPC/DNS failures
3. reduce local load or wait for known bulk changes to drain
4. run repadmin /syncall /AdeP
5. success is a shrinking queue.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [replication-check](./replication-check.md)
- [replication-partner-count](./replication-partner-count.md)
- [replication-consistency](./replication-consistency.md)

## References

- KQL: `packs/ad-ds/docs/kql/replication.kql`
- Bicep: `packs/ad-ds/alerts/replication.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-AdReplicationQueue.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/troubleshoot/troubleshooting-active-directory-replication-problems
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/repadmin
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

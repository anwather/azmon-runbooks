# AD DS — Strict replication consistency

**Alert ID:** `ad-ds-replication-consistency`
**Severity:** 1 (Critical)
**Category:** Replication
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Configuration.ReplicationConsistency.Monitor`

---

## Symptom

The alert description is: "Strict replication consistency is not enabled." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

Strict replication consistency is disabled. Lingering objects can replicate after long outages and cause object resurrection or divergent directory state.

## Common causes

- Registry changed during recovery
- DC restored after extended outage
- Workaround for lingering object errors
- Hardening baseline drift

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1004, 1304, 1604)
| extend CheckId = case(EventID in (1004, 1304, 1604), 4, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
reg query HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Parameters /v "Strict Replication Consistency"
repadmin /regkey $env:COMPUTERNAME +strict
repadmin /showrepl
dcdiag /test:Replications /v
```

## How to fix

1. Enable with repadmin /regkey $env:COMPUTERNAME +strict
2. confirm registry value 1
3. investigate lingering-object errors before forcing sync
4. escalate if offline beyond tombstone lifetime.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [replication-check](./replication-check.md)
- [replication-queue](./replication-queue.md)
- [replication-partner-count](./replication-partner-count.md)

## References

- KQL: `packs/ad-ds/docs/kql/replication.kql`
- Bicep: `packs/ad-ds/alerts/replication.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-AdReplicationConsistency.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/troubleshoot/troubleshooting-active-directory-replication-problems
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/repadmin
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

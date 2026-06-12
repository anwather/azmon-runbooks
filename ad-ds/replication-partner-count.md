# AD DS — Replication partner count

**Alert ID:** `ad-ds-replication-partner-count`
**Severity:** 2 (Error)
**Category:** Replication
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Configuration.ReplicationPartnerCount.Monitor`

---

## Symptom

The alert description is: "AD replication partner count is currently unhealthy." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

The DC has too many replication partners or no valid outbound partners where one is expected. KCC topology may not be keeping the domain converged efficiently.

## Common causes

- KCC topology not converged
- Stale nTDSConnection objects
- Manual connection objects
- Incorrect site links or subnets
- RODC/writable DC role mismatch

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1003, 1303, 1603)
| extend CheckId = case(EventID in (1003, 1303, 1603), 3, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
repadmin /kcc
repadmin /showconn $env:COMPUTERNAME
repadmin /showrepl
Get-ADReplicationConnection -Filter * -Server $env:COMPUTERNAME
```

## How to fix

1. Run repadmin /kcc
2. correct site/subnet/site-link configuration
3. delete stale manual connections only after metadata cleanup
4. success is expected connection count and healthy showrepl.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [replication-check](./replication-check.md)
- [replication-queue](./replication-queue.md)
- [replication-consistency](./replication-consistency.md)

## References

- KQL: `packs/ad-ds/docs/kql/replication.kql`
- Bicep: `packs/ad-ds/alerts/replication.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-AdReplicationPartnerCount.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/troubleshoot/troubleshooting-active-directory-replication-problems
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/repadmin
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

# AD DS — Connection object replication state degraded

**Alert ID:** `ad-ds-connection-objects`
**Severity:** 3 (Warning)
**Category:** Topology
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `ConnectionObject.Monitor`

---

## Symptom

The alert description is: "One or more nTDSConnection objects for this Domain Controller are reporting a non-OK replication state. Mirrors SCOM ConnectionObject.Monitor (checkId 140)." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

One or more nTDSConnection objects report degraded replication state. This usually indicates source-server or topology problems.

## Common causes

- Source DC offline
- KCC routed around failed site link
- Manual/stale connection
- Replication access denied or RPC failure
- Topology changed but KCC not converged

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1140, 1440, 1740)
| extend CheckId = case(EventID in (1140, 1440, 1740), 140, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
repadmin /showconn $env:COMPUTERNAME
repadmin /showrepl $env:COMPUTERNAME
repadmin /kcc
Get-ADReplicationConnection -Filter * -Server $env:COMPUTERNAME
```

## How to fix

1. Identify source connection from payload
2. fix source DC connectivity/replication
3. run repadmin /kcc
4. remove stale manual objects only after metadata cleanup.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- None.

## References

- KQL: `packs/ad-ds/docs/kql/connection-objects.kql`
- Bicep: `packs/ad-ds/alerts/connectionObjects.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-ConnectionObjects.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/plan/active-directory-domain-services-site-topology
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/repadmin
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

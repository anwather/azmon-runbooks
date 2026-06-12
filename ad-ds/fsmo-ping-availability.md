# AD DS — FSMO ping availability

**Alert ID:** `ad-ds-fsmo-ping-availability`
**Severity:** 1 (Critical)
**Category:** FSMO
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Availability.FsmoPing.*.Monitor`

---

## Symptom

The alert description is: "One or more FSMO role holders are not currently reachable by ping." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

A FSMO role holder cannot be reached by ICMP. Role-specific operations such as RID allocation, password changes, schema/domain updates, or time hierarchy can be affected.

## Common causes

- FSMO holder offline
- DNS resolves to stale address
- Firewall or route blocks ICMP
- Recent role transfer not replicated
- Site or VPN outage

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1010, 1310, 1610, 1011, 1311, 1611, 1012, 1312, 1612, 1013, 1313, 1613, 1014, 1314, 1614)
| extend CheckId = case(EventID in (1010, 1310, 1610), 10, EventID in (1011, 1311, 1611), 11, EventID in (1012, 1312, 1612), 12, EventID in (1013, 1313, 1613), 13, EventID in (1014, 1314, 1614), 14, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
netdom query fsmo
$FsmoHolder = "<replace-with-FSMO-holder>"
Resolve-DnsName $FsmoHolder
Test-Connection $FsmoHolder -Count 5
nltest /dsgetdc:$env:USERDNSDOMAIN /force
```

## How to fix

1. Bring the holder online or restore routing
2. fix DNS records
3. confirm Test-Connection succeeds
4. transfer or seize roles only when the holder is unrecoverable and AD owners approve.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [fsmo-bind-availability](./fsmo-bind-availability.md)
- [fsmo-ping-performance](./fsmo-ping-performance.md)
- [fsmo-bind-performance](./fsmo-bind-performance.md)
- [fsmo-consistency](./fsmo-consistency.md)

## References

- KQL: `packs/ad-ds/docs/kql/fsmo-ping-availability.kql`
- Bicep: `packs/ad-ds/alerts/fsmo.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-FsmoPing.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/troubleshoot/windows-server/active-directory/fsmo-roles
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/netdom
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

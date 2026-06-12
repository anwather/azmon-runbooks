# AD DS — FSMO LDAP bind availability

**Alert ID:** `ad-ds-fsmo-bind-availability`
**Severity:** 1 (Critical)
**Category:** FSMO
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Availability.FsmoBind.*.Monitor`

---

## Symptom

The alert description is: "One or more FSMO role holders cannot currently be LDAP-bound." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

A FSMO role holder does not accept LDAP RootDSE binds. Role-dependent operations can fail even if the server pings.

## Common causes

- NTDS stopped or LSASS hung
- LDAP/RPC path blocked
- Kerberos or secure-channel problem
- DNS resolves to wrong host
- Role holder overloaded

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1020, 1320, 1620, 1021, 1321, 1621, 1022, 1322, 1622, 1023, 1323, 1623, 1024, 1324, 1624)
| extend CheckId = case(EventID in (1020, 1320, 1620), 20, EventID in (1021, 1321, 1621), 21, EventID in (1022, 1322, 1622), 22, EventID in (1023, 1323, 1623), 23, EventID in (1024, 1324, 1624), 24, int(null))
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
Get-ADRootDSE -Server $FsmoHolder
nltest /server:$FsmoHolder /sc_query:$env:USERDNSDOMAIN
dcdiag /s:$FsmoHolder /test:Services /test:Advertising
```

## How to fix

1. Start/fix AD DS services on the holder
2. repair LDAP/Kerberos/DNS
3. reduce LSASS load
4. success is Get-ADRootDSE and dcdiag Advertising passing.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [fsmo-ping-availability](./fsmo-ping-availability.md)
- [fsmo-ping-performance](./fsmo-ping-performance.md)
- [fsmo-bind-performance](./fsmo-bind-performance.md)
- [fsmo-consistency](./fsmo-consistency.md)

## References

- KQL: `packs/ad-ds/docs/kql/fsmo-bind-availability.kql`
- Bicep: `packs/ad-ds/alerts/fsmo.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-FsmoBind.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/troubleshoot/windows-server/active-directory/fsmo-roles
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/netdom
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

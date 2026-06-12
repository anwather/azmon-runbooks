# AD DS — FSMO LDAP bind performance

**Alert ID:** `ad-ds-fsmo-bind-performance`
**Severity:** 3 (Warning)
**Category:** FSMO
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Availability.FsmoBind.*.Monitor`

---

## Symptom

The alert description is: "One or more FSMO LDAP bind checks are currently over threshold." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

LDAP bind time to a FSMO holder is above threshold. Slow binds often indicate LSASS pressure, DNS delays, authentication retry, or network latency.

## Common causes

- LSASS CPU/thread pressure
- High-latency site path
- Slow DNS lookup
- Kerberos retries
- Heavy LDAP workload

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
Measure-Command { Get-ADRootDSE -Server $FsmoHolder }
Get-Counter "\Process(lsass)\% Processor Time"
dcdiag /s:$FsmoHolder /test:Advertising
```

## How to fix

1. Fix DNS and secure channel errors
2. reduce LDAP/Kerberos load
3. consider planned FSMO transfer only for persistent holder overload
4. success is bind time below threshold.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [fsmo-ping-availability](./fsmo-ping-availability.md)
- [fsmo-bind-availability](./fsmo-bind-availability.md)
- [fsmo-ping-performance](./fsmo-ping-performance.md)
- [fsmo-consistency](./fsmo-consistency.md)

## References

- KQL: `packs/ad-ds/docs/kql/fsmo-bind-performance.kql`
- Bicep: `packs/ad-ds/alerts/fsmo.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-FsmoBind.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/troubleshoot/windows-server/active-directory/fsmo-roles
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/netdom
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

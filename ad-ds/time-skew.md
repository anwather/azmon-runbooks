# AD DS — Time skew

**Alert ID:** `ad-ds-time-skew`
**Severity:** 3 (Warning)
**Category:** Time
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `TimeSkew.Monitor`

---

## Symptom

The alert description is: "Time skew with the PDC emulator is currently over threshold." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

The DC time differs from the PDC emulator beyond threshold. Kerberos authentication can fail when clocks drift.

## Common causes

- PDC source incorrect
- W32Time stopped/misconfigured
- Hypervisor time override
- NTP blocked
- Manual time change

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1070, 1370, 1670)
| extend CheckId = case(EventID in (1070, 1370, 1670), 70, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
w32tm /query /status
w32tm /query /source
w32tm /monitor
nltest /dsgetdc:$env:USERDNSDOMAIN /pdc
```

## How to fix

1. Return non-PDC DCs to domhier sync
2. restart W32Time and resync
3. configure reliable external peers on PDC
4. success is low offset.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- None.

## References

- KQL: `packs/ad-ds/docs/kql/time-skew.kql`
- Bicep: `packs/ad-ds/alerts/time.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-TimeSkew.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings
- Microsoft docs: https://learn.microsoft.com/windows-server/networking/windows-time-service/how-the-windows-time-service-works
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

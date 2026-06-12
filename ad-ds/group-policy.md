# AD DS — Group Policy update

**Alert ID:** `ad-ds-group-policy`
**Severity:** 3 (Warning)
**Category:** Group Policy
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Configuration.GroupPolicy.Monitor`

---

## Symptom

The alert description is: "Group Policy processing is currently unhealthy." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

Group Policy processing is unhealthy on the DC. Security settings, audit policy, or scripts may be inconsistent.

## Common causes

- SYSVOL unavailable
- DNS/DC locator failure
- Corrupt local policy cache
- GPO access denied or missing
- Group Policy Client/WMI issue

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1100, 1400, 1700)
| extend CheckId = case(EventID in (1100, 1400, 1700), 100, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
gpupdate /force
gpresult /h gpresult.html /scope computer
dcdiag /test:Advertising /v
net view \$env:COMPUTERNAME\SYSVOL
```

## How to fix

1. Fix SYSVOL/DNS first
2. review gpresult for failing GPO
3. repair GPO ACL/content
4. success is gpupdate clean.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- None.

## References

- KQL: `packs/ad-ds/docs/kql/group-policy.kql`
- Bicep: `packs/ad-ds/alerts/groupPolicy.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-GroupPolicy.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/troubleshoot/windows-server/group-policy/applying-group-policy-troubleshooting-guidance
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/gpresult
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

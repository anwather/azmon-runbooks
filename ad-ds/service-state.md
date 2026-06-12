# AD DS — Service state

**Alert ID:** `ad-ds-service-state`
**Severity:** 1 (Critical)
**Category:** Services
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `*.ServiceCheck monitors collapsed`

---

## Symptom

The alert description is: "One or more AD DS related services are currently stopped or missing." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

One or more AD DS related services are stopped or missing. The payload identifies NTDS, KDC, Netlogon, W32Time, DNS, DFSR, ADWS, or another service.

## Common causes

- Service failed after patching
- Role-specific service missing
- Dependency failure
- Disk/database issue
- Manual disablement or baseline drift

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1200, 1500, 1800)
| extend CheckId = case(EventID in (1200, 1500, 1800), 200, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
Get-Service NTDS,KDC,Netlogon,W32Time,IsmServ,DFS,gpsvc,dnscache,ADWS,DNS,DFSR,NtFrs -ErrorAction SilentlyContinue
dcdiag /test:Services /v
wevtutil qe System /c:30 /f:text
$ServiceName = "<service-name-from-alert-payload>"
sc.exe query $ServiceName
```

## How to fix

1. Start stopped service
2. inspect System events if it fails
3. validate role applicability for DNS/DFSR/NtFrs
4. escalate immediately for NTDS/KDC/Netlogon failures.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- None.

## References

- KQL: `packs/ad-ds/docs/kql/service-state.kql`
- Bicep: `packs/ad-ds/alerts/services.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-AdServiceState.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/dcdiag
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

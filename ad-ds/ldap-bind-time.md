# AD DS — LDAP bind time

**Alert ID:** `ad-ds-ldap-bind-time`
**Severity:** 3 (Warning)
**Category:** LDAP/GC
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Performance.BindTimes.Monitor`

---

## Symptom

The alert description is: "Local LDAP bind time is currently over threshold." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

LDAP RootDSE bind succeeds but is slow. This usually indicates LSASS pressure, DNS delay, or authentication retries.

## Common causes

- High LDAP/Kerberos workload
- DNS latency
- CPU/disk contention
- Security software inspection
- Secure-channel retries

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1051, 1351, 1651)
| extend CheckId = case(EventID in (1051, 1351, 1651), 51, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
Measure-Command { Get-ADRootDSE -Server localhost }
Get-Counter "\Process(lsass)\% Processor Time"
dcdiag /test:Advertising /v
nltest /sc_query:$env:USERDNSDOMAIN
```

## How to fix

1. Confirm repeatable latency
2. fix DNS and secure-channel errors
3. redirect heavy LDAP clients
4. success is bind time below threshold.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [ldap-bind-availability](./ldap-bind-availability.md)
- [gc-search-availability](./gc-search-availability.md)
- [gc-search-performance](./gc-search-performance.md)

## References

- KQL: `packs/ad-ds/docs/kql/ldap-bind.kql`
- Bicep: `packs/ad-ds/alerts/ldap-gc.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-LdapBindTime.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/plan/global-catalog
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/component-updates/performance-tuning-active-directory
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

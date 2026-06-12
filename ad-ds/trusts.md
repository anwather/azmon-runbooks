# AD DS — Trust health

**Alert ID:** `ad-ds-trusts`
**Severity:** 2 (Error)
**Category:** Trusts
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `AD_Monitor_Trusts.Monitor`

---

## Symptom

The alert description is: "One or more AD trusts are currently unhealthy." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

One or more AD trusts are unhealthy. Cross-domain or cross-forest authentication and authorization can fail.

## Common causes

- Trust password mismatch
- DNS conditional forwarder failure
- Firewall/RPC/Kerberos blocked
- Clock skew
- Trusted DC unreachable

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1080, 1380, 1680)
| extend CheckId = case(EventID in (1080, 1380, 1680), 80, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
nltest /domain_trusts
$TrustedDomain = "<replace-with-trusted-domain>"
nltest /sc_query:$TrustedDomain
netdom trust $TrustedDomain /domain:$env:USERDNSDOMAIN /verify
dcdiag /test:Trusts /v
```

## How to fix

1. Fix trusted-domain DNS
2. restore Kerberos/LDAP/RPC path
3. reset trust password only with domain owner approval
4. success is nltest and netdom verify passing.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- None.

## References

- KQL: `packs/ad-ds/docs/kql/trusts.kql`
- Bicep: `packs/ad-ds/alerts/trusts.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-AdTrusts.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/netdom-trust
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/plan/forest-design-models
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

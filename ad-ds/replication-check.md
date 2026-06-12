# AD DS — Replication health check

**Alert ID:** `ad-ds-replication-check`
**Severity:** 1 (Critical)
**Category:** Replication
**Detection mechanism:** Probe script
**Source SCOM monitor (if any):** `Availability.ReplicationShowReplCheck.Monitor`

---

## Symptom

The alert description is: "AD replication showrepl freshness or failures are currently unhealthy." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

The affected DC has replication failures or stale last-success timestamps. Authentication may continue locally, but password resets, DNS, GPO, and directory updates can diverge.

## Common causes

- Broken RPC/DNS to partners
- Stale partner after demotion or site change
- Long outage or delayed naming-context replication
- Time skew or secure-channel failure
- Lingering object protection blocking replication

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Event
| where TimeGenerated > ago(24h)
| where Source == "HealthProbe.ad-ds"
| where EventID in (1001, 1301, 1601)
| extend CheckId = case(EventID in (1001, 1301, 1601), 1, int(null))
| extend RawPayload = tostring(column_ifexists("RenderedDescription", ""))
| extend RawPayload = iff(isempty(RawPayload), tostring(column_ifexists("Message", "")), RawPayload)
| extend P = parse_json(RawPayload)
| where tostring(P.severity) != "OK"
| project TimeGenerated, Computer, CheckId, Severity=tostring(P.severity), Message=tostring(P.message), Payload=P.payload
| order by TimeGenerated desc
```

```powershell
repadmin /replsummary
repadmin /showrepl * /csv
dcdiag /test:Replications /v
Get-ADReplicationFailure -Target $env:COMPUTERNAME -Scope Server
```

## How to fix

1. Correct DNS and RPC reachability to failing partners
2. remove stale metadata only after validating demotion
3. run repadmin /syncall /AdeP after repair
4. success is no failed partners and a fresh last-success time.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [replication-queue](./replication-queue.md)
- [replication-partner-count](./replication-partner-count.md)
- [replication-consistency](./replication-consistency.md)

## References

- KQL: `packs/ad-ds/docs/kql/replication.kql`
- Bicep: `packs/ad-ds/alerts/replication.bicep`
- Probe script (if applicable): `scripts/packs/ad-ds/Public/Test-AdReplication.ps1`
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/troubleshoot/troubleshooting-active-directory-replication-problems
- Microsoft docs: https://learn.microsoft.com/windows-server/administration/windows-commands/repadmin
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

# AD DS — LSASS CPU

**Alert ID:** `ad-ds-performance-lsass-cpu`
**Severity:** 2 (Error)
**Category:** Performance
**Detection mechanism:** Perf counter threshold
**Source SCOM monitor (if any):** `PerformanceEssentialServices.LSASS.Monitor`

---

## Symptom

The alert description is: "LSASS process CPU is over threshold." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

LSASS CPU is above threshold. Sustained pressure slows authentication, LDAP, GC, and replication on the DC.

## Common causes

- Authentication or LDAP spike
- Expensive LDAP filters
- Replication storm
- Security software interaction
- Undersized DC or site mapping issue

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Perf
| where TimeGenerated > ago(24h)
| where ObjectName == "Process" and CounterName == "% Processor Time"
| where tolower(InstanceName) == "lsass"
| summarize AvgCpu=avg(CounterValue), MaxCpu=max(CounterValue) by Computer, bin(TimeGenerated, 5m)
| where AvgCpu > 80
| order by TimeGenerated desc
```

```powershell
Get-Counter "\Process(lsass)\% Processor Time"
Get-Counter "\NTDS\LDAP Searches/sec"
Get-Counter "\NTDS\Kerberos Authentications/sec"
dcdiag /test:Services /v
```

## How to fix

1. Classify load as LDAP/Kerberos/replication
2. move clients or fix site mappings
3. tune application LDAP
4. scale DC CPU/capacity.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [performance-atq](./performance-atq.md)
- [performance-dns-process-cpu](./performance-dns-process-cpu.md)

## References

- KQL: `packs/ad-ds/docs/kql/performance-lsass.kql`
- Bicep: `packs/ad-ds/alerts/performance.bicep`
- Probe script (if applicable): none; this alert uses collected Windows events or native perf counters.
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/component-updates/performance-tuning-active-directory
- Microsoft docs: https://learn.microsoft.com/troubleshoot/windows-server/identity/high-lsassexe-cpu-utilization-troubleshooting
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

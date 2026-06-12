# AD DS — DNS process CPU

**Alert ID:** `ad-ds-performance-dns-process-cpu`
**Severity:** 3 (Warning)
**Category:** Performance
**Detection mechanism:** Perf counter threshold
**Source SCOM monitor (if any):** `PerformanceEssentialServices.DNS.Monitor`

---

## Symptom

The alert description is: "DNS process CPU is over threshold." Treat the affected `Computer` in the alert results as the starting DC for diagnosis.

## What it means

DNS.exe CPU is above threshold. Slow DNS affects DC locator, Kerberos/LDAP discovery, and name resolution.

## Common causes

- High query or recursion volume
- Scavenging/zone loading
- Unreachable forwarders
- AD-integrated zone churn
- Dynamic update storm

## How to diagnose

Start with the 24-hour Log Analytics query, then run the command sequence on the affected DC. Probe payloads normally identify the target role, partner, service, measured value, threshold, and exception text needed for triage.

```kusto
Perf
| where TimeGenerated > ago(24h)
| where ObjectName == "Process" and CounterName == "% Processor Time"
| where tolower(InstanceName) == "dns"
| summarize AvgCpu=avg(CounterValue), MaxCpu=max(CounterValue) by Computer, bin(TimeGenerated, 5m)
| where AvgCpu > 80
| order by TimeGenerated desc
```

```powershell
Get-Counter "\Process(dns)\% Processor Time"
dnscmd /statistics
dcdiag /test:DNS /v
Get-DnsServerForwarder
```

## How to fix

1. Confirm sustained CPU
2. fix unreachable forwarders
3. remediate noisy clients/dynamic updates
4. add DNS/DC capacity if legitimate load is high.

## Escalation

Escalate to domain owners or AD engineering if the issue affects multiple DCs, persists after the listed fix, requires FSMO seizure, RID exhaustion handling, lingering-object cleanup, database repair, authoritative SYSVOL restore, or any action that could remove or rewrite directory data. Engage networking when evidence points to packet loss, blocked ports, unreachable peers, or site routing rather than a local DC fault.

## Related alerts

- [performance-atq](./performance-atq.md)
- [performance-lsass-cpu](./performance-lsass-cpu.md)

## References

- KQL: `packs/ad-ds/docs/kql/performance-dns-process.kql`
- Bicep: `packs/ad-ds/alerts/performance.bicep`
- Probe script (if applicable): none; this alert uses collected Windows events or native perf counters.
- Threshold defaults: `packs/ad-ds/parameters/thresholds.example.json`
- Microsoft docs: https://learn.microsoft.com/windows-server/identity/ad-ds/manage/component-updates/performance-tuning-active-directory
- Microsoft docs: https://learn.microsoft.com/troubleshoot/windows-server/identity/high-lsassexe-cpu-utilization-troubleshooting
- SCOM-era context (optional, when relevant): Kevin Holman blog at https://kevinholman.com

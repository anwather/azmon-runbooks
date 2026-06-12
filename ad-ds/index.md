# AD DS alert runbooks

Standalone operator runbooks for the `ad-ds` Azure Monitor pack.

## Alphabetical index

| Alert | Severity | Category | One-liner |
|---|---|---|---|
| [connection-objects](./connection-objects.md) | 3 (Warning) | Topology | Check nTDSConnection state, source DC reachability, and KCC topology. |
| [database-edb-log-disk](./database-edb-log-disk.md) | 2 (Error) | Database | Free space on the AD log volume or move/resize the log volume. |
| [database-ntds-dit-disk](./database-ntds-dit-disk.md) | 2 (Error) | Database | Free space on the database volume or move/resize NTDS.dit storage. |
| [dc-locator](./dc-locator.md) | 1 (Critical) | SYSVOL/DC Locator | Verify DNS SRV records, Netlogon, and DC reachability from the affected DC. |
| [dfsr-wmi](./dfsr-wmi.md) | 2 (Error) | SYSVOL/DC Locator | Restart/repair DFSR WMI provider and confirm Event 6102 clears Event 6104. |
| [dns-adapters](./dns-adapters.md) | 2 (Error) | DNS | Correct NIC DNS server configuration and reachability. |
| [event-collection-dnsapi](./event-collection-dnsapi.md) | 2 (Error) | Event collection | Validate dynamic update permissions, DNS server reachability, and zone health. |
| [event-collection-gpo-failures](./event-collection-gpo-failures.md) | 2 (Error) | Event collection | Inspect affected GroupPolicy event details and SYSVOL/DNS connectivity. |
| [event-collection-kdc](./event-collection-kdc.md) | 1 (Critical) | Event collection | Review KDC event details for trust, policy, or credential corruption. |
| [event-collection-netlogon-dns](./event-collection-netlogon-dns.md) | 2 (Error) | Event collection | Verify Netlogon, DNS registration, SRV records, and site coverage. |
| [event-collection-rid-pool](./event-collection-rid-pool.md) | 1 (Critical) | Event collection | Check RID Master health and account creation failures. |
| [event-collection-time](./event-collection-time.md) | 3 (Warning) | Event collection | Check W32Time source, hierarchy, and synchronization status. |
| [fsmo-bind-availability](./fsmo-bind-availability.md) | 1 (Critical) | FSMO | Verify the role holder is online and accepts LDAP RootDSE binds. |
| [fsmo-bind-performance](./fsmo-bind-performance.md) | 3 (Warning) | FSMO | ⚠ Currently duplicates bind availability; verify role holder LDAP responsiveness manually. |
| [fsmo-consistency](./fsmo-consistency.md) | 3 (Warning) | FSMO | Compare local role holder view to the PDC and investigate replication. |
| [fsmo-ping-availability](./fsmo-ping-availability.md) | 1 (Critical) | FSMO | Confirm FSMO holder name resolution, network path, and host availability. |
| [fsmo-ping-performance](./fsmo-ping-performance.md) | 3 (Warning) | FSMO | ⚠ Currently duplicates ping availability; test latency manually. |
| [gc-search-availability](./gc-search-availability.md) | 1 (Critical) | LDAP/GC | Confirm the DC is a GC and can search GC://RootDSE. |
| [gc-search-performance](./gc-search-performance.md) | 3 (Warning) | LDAP/GC | Investigate GC query latency, LSASS load, and network delays. |
| [group-policy](./group-policy.md) | 3 (Warning) | Group Policy | Run gpupdate/gpresult on the DC and inspect policy processing errors. |
| [ldap-bind-availability](./ldap-bind-availability.md) | 1 (Critical) | LDAP/GC | Validate local AD DS service, DNS, and LDAP RootDSE bind. |
| [ldap-bind-time](./ldap-bind-time.md) | 3 (Warning) | LDAP/GC | Check LSASS load, LDAP clients, and network latency. |
| [lost-and-found](./lost-and-found.md) | 3 (Warning) | Lost & Found | Inspect CN=LostAndFound for orphaned objects and replication issues. |
| [performance-atq](./performance-atq.md) | 3 (Warning) | Performance | Check LDAP load, thread pool saturation, LSASS CPU, and slow clients. |
| [performance-dns-process-cpu](./performance-dns-process-cpu.md) | 3 (Warning) | Performance | Inspect DNS query volume, scavenging, forwarding, and DNS service health. |
| [performance-lsass-cpu](./performance-lsass-cpu.md) | 2 (Error) | Performance | Identify high-cost LDAP/Kerberos/replication activity and CPU saturation. |
| [replication-check](./replication-check.md) | 1 (Critical) | Replication | Run repadmin /showrepl and fix stale or failed partners. |
| [replication-consistency](./replication-consistency.md) | 1 (Critical) | Replication | Enable strict replication consistency and investigate why it changed. |
| [replication-partner-count](./replication-partner-count.md) | 2 (Error) | Replication | Check KCC topology, site links, RODC status, and connection objects. |
| [replication-queue](./replication-queue.md) | 2 (Error) | Replication | Investigate backlog, slow partners, and DRA pending operations. |
| [rid-pool](./rid-pool.md) | 2 (Error) | RID Pool | Check RID Master health, RID pool free percentage, and account creation risk. |
| [service-state](./service-state.md) | 1 (Critical) | Services | Start missing/stopped AD DS dependent services or validate role applicability. |
| [sysvol-share](./sysvol-share.md) | 1 (Critical) | SYSVOL/DC Locator | Confirm SYSVOL is shared and DFSR/FRS SYSVOL replication is healthy. |
| [time-skew](./time-skew.md) | 3 (Warning) | Time | Compare local time to PDC emulator and repair W32Time configuration. |
| [trusts](./trusts.md) | 2 (Error) | Trusts | Validate trust status, secure channel, DNS, and cross-domain connectivity. |

## Grouped by category

### Replication
- [replication-check](./replication-check.md) — Run repadmin /showrepl and fix stale or failed partners.
- [replication-queue](./replication-queue.md) — Investigate backlog, slow partners, and DRA pending operations.
- [replication-partner-count](./replication-partner-count.md) — Check KCC topology, site links, RODC status, and connection objects.
- [replication-consistency](./replication-consistency.md) — Enable strict replication consistency and investigate why it changed.

### FSMO
- [fsmo-ping-availability](./fsmo-ping-availability.md) — Confirm FSMO holder name resolution, network path, and host availability.
- [fsmo-bind-availability](./fsmo-bind-availability.md) — Verify the role holder is online and accepts LDAP RootDSE binds.
- [fsmo-ping-performance](./fsmo-ping-performance.md) — ⚠ Currently duplicates ping availability; test latency manually.
- [fsmo-bind-performance](./fsmo-bind-performance.md) — ⚠ Currently duplicates bind availability; verify role holder LDAP responsiveness manually.
- [fsmo-consistency](./fsmo-consistency.md) — Compare local role holder view to the PDC and investigate replication.

### Database
- [database-ntds-dit-disk](./database-ntds-dit-disk.md) — Free space on the database volume or move/resize NTDS.dit storage.
- [database-edb-log-disk](./database-edb-log-disk.md) — Free space on the AD log volume or move/resize the log volume.

### LDAP/GC
- [ldap-bind-availability](./ldap-bind-availability.md) — Validate local AD DS service, DNS, and LDAP RootDSE bind.
- [ldap-bind-time](./ldap-bind-time.md) — Check LSASS load, LDAP clients, and network latency.
- [gc-search-availability](./gc-search-availability.md) — Confirm the DC is a GC and can search GC://RootDSE.
- [gc-search-performance](./gc-search-performance.md) — Investigate GC query latency, LSASS load, and network delays.

### Performance
- [performance-atq](./performance-atq.md) — Check LDAP load, thread pool saturation, LSASS CPU, and slow clients.
- [performance-lsass-cpu](./performance-lsass-cpu.md) — Identify high-cost LDAP/Kerberos/replication activity and CPU saturation.
- [performance-dns-process-cpu](./performance-dns-process-cpu.md) — Inspect DNS query volume, scavenging, forwarding, and DNS service health.

### Time
- [time-skew](./time-skew.md) — Compare local time to PDC emulator and repair W32Time configuration.

### Trusts
- [trusts](./trusts.md) — Validate trust status, secure channel, DNS, and cross-domain connectivity.

### DNS
- [dns-adapters](./dns-adapters.md) — Correct NIC DNS server configuration and reachability.

### Group Policy
- [group-policy](./group-policy.md) — Run gpupdate/gpresult on the DC and inspect policy processing errors.

### Lost & Found
- [lost-and-found](./lost-and-found.md) — Inspect CN=LostAndFound for orphaned objects and replication issues.

### SYSVOL/DC Locator
- [sysvol-share](./sysvol-share.md) — Confirm SYSVOL is shared and DFSR/FRS SYSVOL replication is healthy.
- [dc-locator](./dc-locator.md) — Verify DNS SRV records, Netlogon, and DC reachability from the affected DC.
- [dfsr-wmi](./dfsr-wmi.md) — Restart/repair DFSR WMI provider and confirm Event 6102 clears Event 6104.

### Services
- [service-state](./service-state.md) — Start missing/stopped AD DS dependent services or validate role applicability.

### Topology
- [connection-objects](./connection-objects.md) — Check nTDSConnection state, source DC reachability, and KCC topology.

### RID Pool
- [rid-pool](./rid-pool.md) — Check RID Master health, RID pool free percentage, and account creation risk.

### Event collection
- [event-collection-kdc](./event-collection-kdc.md) — Review KDC event details for trust, policy, or credential corruption.
- [event-collection-netlogon-dns](./event-collection-netlogon-dns.md) — Verify Netlogon, DNS registration, SRV records, and site coverage.
- [event-collection-rid-pool](./event-collection-rid-pool.md) — Check RID Master health and account creation failures.
- [event-collection-gpo-failures](./event-collection-gpo-failures.md) — Inspect affected GroupPolicy event details and SYSVOL/DNS connectivity.
- [event-collection-time](./event-collection-time.md) — Check W32Time source, hierarchy, and synchronization status.
- [event-collection-dnsapi](./event-collection-dnsapi.md) — Validate dynamic update permissions, DNS server reachability, and zone health.

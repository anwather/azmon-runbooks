# SQL Server v1 alert runbooks

Per-alert troubleshooting knowledge base for the proposed SQL Server v1 Azure Monitor pack. Alerts are sourced from the feasibility design, enabled UnitMonitor mapping, Windows Application Event rules, and the selected high-value SQL ERRORLOG buckets.

## Alphabetical index

| Alert | Severity | Category | One-liner |
|---|---|---|---|
| [sql-ag-automatic-failover-readiness](./sql-ag-automatic-failover-readiness.md) | 2 Error | AlwaysOn | Automatic failover may not occur during a primary outage because the secondary is not synchronized, not healthy, or not configured for automatic failover. |
| [sql-ag-online](./sql-ag-online.md) | 2 Error | AlwaysOn | The AG is not fully online from the primary perspective. |
| [sql-ag-replica-role-state](./sql-ag-replica-role-state.md) | 3 Warning | AlwaysOn | A replica can be RESOLVING or UNKNOWN during failover, cluster transitions, or quorum loss. |
| [sql-ag-replicas-connected](./sql-ag-replicas-connected.md) | 3 Warning | AlwaysOn | AG data movement requires connected endpoints. |
| [sql-ag-replicas-data-sync](./sql-ag-replicas-data-sync.md) | 3 Warning | AlwaysOn | Data movement is delayed or suspended for at least one replica/database. |
| [sql-ag-sync-replicas-data-sync](./sql-ag-sync-replicas-data-sync.md) | 3 Warning | AlwaysOn | Synchronous replicas must reach SYNCHRONIZED for zero-data-loss failover. |
| [sql-ag-user-policy-error](./sql-ag-user-policy-error.md) | 2 Error | AlwaysOn | The customer-defined AG policy has failed. |
| [sql-ag-user-policy-warning](./sql-ag-user-policy-warning.md) | 3 Warning | AlwaysOn | The customer-defined AG warning policy has failed. |
| [sql-agent-alert-engine-stopped](./sql-agent-alert-engine-stopped.md) | 1 Critical | Agent | Agent may continue running jobs, but alert processing is stopped because local event log access is broken. |
| [sql-agent-could-not-start](./sql-agent-could-not-start.md) | 1 Critical | Agent | Agent failed during startup before it could schedule jobs or process alerts. |
| [sql-agent-job-duration](./sql-agent-job-duration.md) | 2 Error | Agent | The job may be blocked, waiting on I/O, processing unusually large data, hung in an external subsystem, or running with an unrealistic threshold. |
| [sql-agent-job-failed](./sql-agent-job-failed.md) | 2 Error | Agent | A scheduled job failed. |
| [sql-agent-job-last-run](./sql-agent-job-last-run.md) | 3 Warning | Agent | The last scheduled or manual execution failed. |
| [sql-agent-job-step-exception](./sql-agent-job-step-exception.md) | 1 Critical | Agent | The subsystem process or provider raised an exception while running a job step. |
| [sql-agent-job-step-subsystem-load-fail](./sql-agent-job-step-subsystem-load-fail.md) | 1 Critical | Agent | Agent could not initialize a subsystem such as CmdExec, PowerShell, SSIS, Replication, Analysis Services, or T-SQL proxy support. |
| [sql-agent-self-termination](./sql-agent-self-termination.md) | 1 Critical | Agent | Agent is shutting itself down because an unrecoverable condition was detected. |
| [sql-agent-service-status](./sql-agent-service-status.md) | 2 Error | Agent | Scheduled jobs, alerts, operators, replication agents, log shipping jobs, and maintenance plans may not run until Agent is running. |
| [sql-agent-unable-to-connect](./sql-agent-unable-to-connect.md) | 1 Critical | Agent | Agent cannot connect to the Database Engine instance it manages. |
| [sql-agent-unable-to-reopen-eventlog](./sql-agent-unable-to-reopen-eventlog.md) | 1 Critical | Agent | Agent cannot write/read Application log records that it uses for alerts and diagnostics. |
| [sql-database-log-shipping](./sql-database-log-shipping.md) | 2 Error | Database | The log-shipping restore chain is delayed or broken. |
| [sql-database-rows-size-percent](./sql-database-rows-size-percent.md) | 3 Warning | Storage | Data files or their hosting volumes are running out of usable space. |
| [sql-database-securables-config](./sql-database-securables-config.md) | 3 Warning | Security | The database is online but permissions prevent reliable per-database monitoring. |
| [sql-database-status](./sql-database-status.md) | 2 Error | Database | SQL cannot make the database normally available. |
| [sql-database-tx-log-space](./sql-database-tx-log-space.md) | 2 Error | Storage | The log cannot truncate or grow enough for current workload. |
| [sql-database-user-policy-error](./sql-database-user-policy-error.md) | 2 Error | Database | The customer-defined database policy has failed. |
| [sql-database-user-policy-warning](./sql-database-user-policy-warning.md) | 3 Warning | Database | The customer-defined database warning policy has failed. |
| [sql-db-replica-data-sync](./sql-db-replica-data-sync.md) | 3 Warning | AlwaysOn | A database replica is behind, suspended, or not joined. |
| [sql-db-replica-error-policy-state](./sql-db-replica-error-policy-state.md) | 2 Error | AlwaysOn | The customer-defined DB replica error policy has failed. |
| [sql-db-replica-suspend](./sql-db-replica-suspend.md) | 3 Warning | AlwaysOn | The AG database is not sending or applying log records. |
| [sql-db-replica-warning-policy-state](./sql-db-replica-warning-policy-state.md) | 3 Warning | AlwaysOn | The customer-defined DB replica warning policy has failed and may indicate a configuration drift or degraded replica condition. |
| [sql-engine-average-wait-time](./sql-engine-average-wait-time.md) | 3 Warning | Performance | SQL workers are spending too much time waiting on resources such as storage, locks, memory grants, CPU schedulers, or network I/O. |
| [sql-engine-cpu-utilization](./sql-engine-cpu-utilization.md) | 2 Error | Performance | Schedulers are saturated or close to saturation. |
| [sql-engine-disk-latency-read](./sql-engine-disk-latency-read.md) | 2 Error | Storage | Data and log reads are taking too long at the Windows storage layer. |
| [sql-engine-disk-latency-write](./sql-engine-disk-latency-write.md) | 2 Error | Storage | Log and data writes are slow. |
| [sql-engine-securables-config](./sql-engine-securables-config.md) | 3 Warning | Security | The probe is not seeing enough metadata to reliably monitor SQL. |
| [sql-engine-service-pack-level](./sql-engine-service-pack-level.md) | 3 Warning | Engine Service | The instance is missing a required service pack, cumulative update, or GDR. |
| [sql-engine-service-restart-mssqlserver](./sql-engine-service-restart-mssqlserver.md) | 1 Critical | Engine Service | The default instance restarted. |
| [sql-engine-service-restart-named-instance](./sql-engine-service-restart-named-instance.md) | 1 Critical | Engine Service | The named instance restarted. |
| [sql-engine-service-status](./sql-engine-service-status.md) | 2 Error | Engine Service | Engine service state for the monitored instance is stopped, paused, or cannot be queried. |
| [sql-engine-status](./sql-engine-status.md) | 2 Error | Engine Service | The monitoring login cannot complete a basic connection and metadata query. |
| [sql-engine-stolen-server-memory](./sql-engine-stolen-server-memory.md) | 2 Error | Performance | Memory consumers such as query workspace, plan cache, locks, CLR, columnstore, or in-memory features are taking memory that could otherwise support data cache. |
| [sql-engine-thread-count](./sql-engine-thread-count.md) | 2 Error | Performance | Worker starvation is possible. |
| [sql-engine-wmi-health](./sql-engine-wmi-health.md) | 2 Error | Engine Service | SQL Server monitoring cannot read the ComputerManagement WMI provider. |
| [sql-error-1101-allocate-failed-tempdb](./sql-error-1101-allocate-failed-tempdb.md) | 2 Error | Storage | SQL could not allocate database pages. |
| [sql-error-1105-disk-full](./sql-error-1105-disk-full.md) | 2 Error | Storage | SQL could not allocate database pages. |
| [sql-error-1418-mirroring-endpoint](./sql-error-1418-mirroring-endpoint.md) | 2 Error | AlwaysOn | AlwaysOn availability, data movement, or WSFC coordination is impaired. |
| [sql-error-17066-sql-assertion](./sql-error-17066-sql-assertion.md) | 2 Error | ERRORLOG | SQL Server hit an internal assertion. |
| [sql-error-18204-backup-failed-device](./sql-error-18204-backup-failed-device.md) | 2 Error | ERRORLOG | A backup or restore operation could not access its media or complete. |
| [sql-error-18210-backup-failed-device-other](./sql-error-18210-backup-failed-device-other.md) | 2 Error | ERRORLOG | A backup or restore operation could not access its media or complete. |
| [sql-error-3041-backup-failed](./sql-error-3041-backup-failed.md) | 2 Error | ERRORLOG | A backup or restore operation could not access its media or complete. |
| [sql-error-3201-cannot-open-backup-device](./sql-error-3201-cannot-open-backup-device.md) | 2 Error | ERRORLOG | A backup or restore operation could not access its media or complete. |
| [sql-error-33108-tde-key-failure](./sql-error-33108-tde-key-failure.md) | 1 Critical | Security | SQL cannot access TDE key material required to decrypt or protect the database encryption key. |
| [sql-error-33111-tde-cert-failure](./sql-error-33111-tde-cert-failure.md) | 1 Critical | Security | SQL cannot access TDE key material required to decrypt or protect the database encryption key. |
| [sql-error-35262-ag-replica-down](./sql-error-35262-ag-replica-down.md) | 1 Critical | AlwaysOn | AlwaysOn availability, data movement, or WSFC coordination is impaired. |
| [sql-error-35264-ag-data-movement-suspended](./sql-error-35264-ag-data-movement-suspended.md) | 2 Error | AlwaysOn | AlwaysOn availability, data movement, or WSFC coordination is impaired. |
| [sql-error-35275-ag-lease-expired](./sql-error-35275-ag-lease-expired.md) | 1 Critical | AlwaysOn | AlwaysOn availability, data movement, or WSFC coordination is impaired. |
| [sql-error-41048-ag-cluster-lost-quorum](./sql-error-41048-ag-cluster-lost-quorum.md) | 1 Critical | AlwaysOn | AlwaysOn availability, data movement, or WSFC coordination is impaired. |
| [sql-error-41097-ag-availability-failed](./sql-error-41097-ag-availability-failed.md) | 1 Critical | AlwaysOn | AlwaysOn availability, data movement, or WSFC coordination is impaired. |
| [sql-error-41131-ag-failed-to-bring-online](./sql-error-41131-ag-failed-to-bring-online.md) | 1 Critical | AlwaysOn | AlwaysOn availability, data movement, or WSFC coordination is impaired. |
| [sql-error-823-io-logical-error](./sql-error-823-io-logical-error.md) | 1 Critical | Storage | SQL has evidence of unreliable storage or page-level corruption. |
| [sql-error-824-io-logical-checksum](./sql-error-824-io-logical-checksum.md) | 1 Critical | Storage | SQL has evidence of unreliable storage or page-level corruption. |
| [sql-error-825-read-retry](./sql-error-825-read-retry.md) | 3 Warning | Storage | SQL has evidence of unreliable storage or page-level corruption. |
| [sql-error-832-constant-page-failure](./sql-error-832-constant-page-failure.md) | 1 Critical | Storage | SQL has evidence of unreliable storage or page-level corruption. |
| [sql-error-8928-page-corruption](./sql-error-8928-page-corruption.md) | 1 Critical | ERRORLOG | DBCC or SQL detected physical/logical consistency errors. |
| [sql-error-8939-table-error](./sql-error-8939-table-error.md) | 1 Critical | ERRORLOG | DBCC or SQL detected physical/logical consistency errors. |
| [sql-error-9002-log-full](./sql-error-9002-log-full.md) | 2 Error | Storage | The transaction log cannot reuse or grow enough space. |
| [sql-error-9689-sb-manager-shutdown](./sql-error-9689-sb-manager-shutdown.md) | 2 Error | Service Broker | Service Broker infrastructure is not running correctly. |
| [sql-error-9694-sb-cannot-start](./sql-error-9694-sb-cannot-start.md) | 2 Error | Service Broker | Service Broker infrastructure is not running correctly. |
| [sql-error-9695-sb-task-mgr-memory](./sql-error-9695-sb-task-mgr-memory.md) | 2 Error | Service Broker | Service Broker infrastructure is not running correctly. |
| [sql-error-9696-sb-event-handler-cannot-start](./sql-error-9696-sb-event-handler-cannot-start.md) | 2 Error | Service Broker | Service Broker infrastructure is not running correctly. |
| [sql-error-9697-sb-cannot-start-on-database](./sql-error-9697-sb-cannot-start-on-database.md) | 2 Error | Service Broker | Service Broker infrastructure is not running correctly. |
| [sql-error-9698-sb-security-mgr-cannot-start](./sql-error-9698-sb-security-mgr-cannot-start.md) | 2 Error | Service Broker | Service Broker infrastructure is not running correctly. |
| [sql-replica-data-sync-health](./sql-replica-data-sync-health.md) | 3 Warning | AlwaysOn | One or more databases on this replica are not synchronizing as expected, often because data movement is suspended, the endpoint is disconnected, or redo/send queues are growing. |
| [sql-replica-error-policy-state](./sql-replica-error-policy-state.md) | 2 Error | AlwaysOn | The customer-defined replica policy has failed. |
| [sql-replica-is-connected](./sql-replica-is-connected.md) | 2 Error | AlwaysOn | The local replica cannot communicate with its partner over the database mirroring endpoint. |
| [sql-replica-role-healthy](./sql-replica-role-healthy.md) | 2 Error | AlwaysOn | The replica may be resolving after failover, affected by WSFC quorum, or unable to join the AG role. |
| [sql-replica-warning-policy-state](./sql-replica-warning-policy-state.md) | 3 Warning | AlwaysOn | The customer-defined replica warning policy has failed. |
| [sql-user-resource-pool-memory](./sql-user-resource-pool-memory.md) | 2 Error | Performance | Queries assigned to the pool may experience memory grant waits, spills, or failures. |

## Grouped by category

### Agent

- [sql-agent-alert-engine-stopped](./sql-agent-alert-engine-stopped.md) — Agent may continue running jobs, but alert processing is stopped because local event log access is broken.
- [sql-agent-could-not-start](./sql-agent-could-not-start.md) — Agent failed during startup before it could schedule jobs or process alerts.
- [sql-agent-job-duration](./sql-agent-job-duration.md) — The job may be blocked, waiting on I/O, processing unusually large data, hung in an external subsystem, or running with an unrealistic threshold.
- [sql-agent-job-failed](./sql-agent-job-failed.md) — A scheduled job failed.
- [sql-agent-job-last-run](./sql-agent-job-last-run.md) — The last scheduled or manual execution failed.
- [sql-agent-job-step-exception](./sql-agent-job-step-exception.md) — The subsystem process or provider raised an exception while running a job step.
- [sql-agent-job-step-subsystem-load-fail](./sql-agent-job-step-subsystem-load-fail.md) — Agent could not initialize a subsystem such as CmdExec, PowerShell, SSIS, Replication, Analysis Services, or T-SQL proxy support.
- [sql-agent-self-termination](./sql-agent-self-termination.md) — Agent is shutting itself down because an unrecoverable condition was detected.
- [sql-agent-service-status](./sql-agent-service-status.md) — Scheduled jobs, alerts, operators, replication agents, log shipping jobs, and maintenance plans may not run until Agent is running.
- [sql-agent-unable-to-connect](./sql-agent-unable-to-connect.md) — Agent cannot connect to the Database Engine instance it manages.
- [sql-agent-unable-to-reopen-eventlog](./sql-agent-unable-to-reopen-eventlog.md) — Agent cannot write/read Application log records that it uses for alerts and diagnostics.

### AlwaysOn

- [sql-ag-automatic-failover-readiness](./sql-ag-automatic-failover-readiness.md) — Automatic failover may not occur during a primary outage because the secondary is not synchronized, not healthy, or not configured for automatic failover.
- [sql-ag-online](./sql-ag-online.md) — The AG is not fully online from the primary perspective.
- [sql-ag-replica-role-state](./sql-ag-replica-role-state.md) — A replica can be RESOLVING or UNKNOWN during failover, cluster transitions, or quorum loss.
- [sql-ag-replicas-connected](./sql-ag-replicas-connected.md) — AG data movement requires connected endpoints.
- [sql-ag-replicas-data-sync](./sql-ag-replicas-data-sync.md) — Data movement is delayed or suspended for at least one replica/database.
- [sql-ag-sync-replicas-data-sync](./sql-ag-sync-replicas-data-sync.md) — Synchronous replicas must reach SYNCHRONIZED for zero-data-loss failover.
- [sql-ag-user-policy-error](./sql-ag-user-policy-error.md) — The customer-defined AG policy has failed.
- [sql-ag-user-policy-warning](./sql-ag-user-policy-warning.md) — The customer-defined AG warning policy has failed.
- [sql-db-replica-data-sync](./sql-db-replica-data-sync.md) — A database replica is behind, suspended, or not joined.
- [sql-db-replica-error-policy-state](./sql-db-replica-error-policy-state.md) — The customer-defined DB replica error policy has failed.
- [sql-db-replica-suspend](./sql-db-replica-suspend.md) — The AG database is not sending or applying log records.
- [sql-db-replica-warning-policy-state](./sql-db-replica-warning-policy-state.md) — The customer-defined DB replica warning policy has failed and may indicate a configuration drift or degraded replica condition.
- [sql-error-1418-mirroring-endpoint](./sql-error-1418-mirroring-endpoint.md) — AlwaysOn availability, data movement, or WSFC coordination is impaired.
- [sql-error-35262-ag-replica-down](./sql-error-35262-ag-replica-down.md) — AlwaysOn availability, data movement, or WSFC coordination is impaired.
- [sql-error-35264-ag-data-movement-suspended](./sql-error-35264-ag-data-movement-suspended.md) — AlwaysOn availability, data movement, or WSFC coordination is impaired.
- [sql-error-35275-ag-lease-expired](./sql-error-35275-ag-lease-expired.md) — AlwaysOn availability, data movement, or WSFC coordination is impaired.
- [sql-error-41048-ag-cluster-lost-quorum](./sql-error-41048-ag-cluster-lost-quorum.md) — AlwaysOn availability, data movement, or WSFC coordination is impaired.
- [sql-error-41097-ag-availability-failed](./sql-error-41097-ag-availability-failed.md) — AlwaysOn availability, data movement, or WSFC coordination is impaired.
- [sql-error-41131-ag-failed-to-bring-online](./sql-error-41131-ag-failed-to-bring-online.md) — AlwaysOn availability, data movement, or WSFC coordination is impaired.
- [sql-replica-data-sync-health](./sql-replica-data-sync-health.md) — One or more databases on this replica are not synchronizing as expected, often because data movement is suspended, the endpoint is disconnected, or redo/send queues are growing.
- [sql-replica-error-policy-state](./sql-replica-error-policy-state.md) — The customer-defined replica policy has failed.
- [sql-replica-is-connected](./sql-replica-is-connected.md) — The local replica cannot communicate with its partner over the database mirroring endpoint.
- [sql-replica-role-healthy](./sql-replica-role-healthy.md) — The replica may be resolving after failover, affected by WSFC quorum, or unable to join the AG role.
- [sql-replica-warning-policy-state](./sql-replica-warning-policy-state.md) — The customer-defined replica warning policy has failed.

### Database

- [sql-database-log-shipping](./sql-database-log-shipping.md) — The log-shipping restore chain is delayed or broken.
- [sql-database-status](./sql-database-status.md) — SQL cannot make the database normally available.
- [sql-database-user-policy-error](./sql-database-user-policy-error.md) — The customer-defined database policy has failed.
- [sql-database-user-policy-warning](./sql-database-user-policy-warning.md) — The customer-defined database warning policy has failed.

### ERRORLOG

- [sql-error-17066-sql-assertion](./sql-error-17066-sql-assertion.md) — SQL Server hit an internal assertion.
- [sql-error-18204-backup-failed-device](./sql-error-18204-backup-failed-device.md) — A backup or restore operation could not access its media or complete.
- [sql-error-18210-backup-failed-device-other](./sql-error-18210-backup-failed-device-other.md) — A backup or restore operation could not access its media or complete.
- [sql-error-3041-backup-failed](./sql-error-3041-backup-failed.md) — A backup or restore operation could not access its media or complete.
- [sql-error-3201-cannot-open-backup-device](./sql-error-3201-cannot-open-backup-device.md) — A backup or restore operation could not access its media or complete.
- [sql-error-8928-page-corruption](./sql-error-8928-page-corruption.md) — DBCC or SQL detected physical/logical consistency errors.
- [sql-error-8939-table-error](./sql-error-8939-table-error.md) — DBCC or SQL detected physical/logical consistency errors.

### Engine Service

- [sql-engine-service-pack-level](./sql-engine-service-pack-level.md) — The instance is missing a required service pack, cumulative update, or GDR.
- [sql-engine-service-restart-mssqlserver](./sql-engine-service-restart-mssqlserver.md) — The default instance restarted.
- [sql-engine-service-restart-named-instance](./sql-engine-service-restart-named-instance.md) — The named instance restarted.
- [sql-engine-service-status](./sql-engine-service-status.md) — Engine service state for the monitored instance is stopped, paused, or cannot be queried.
- [sql-engine-status](./sql-engine-status.md) — The monitoring login cannot complete a basic connection and metadata query.
- [sql-engine-wmi-health](./sql-engine-wmi-health.md) — SQL Server monitoring cannot read the ComputerManagement WMI provider.

### Performance

- [sql-engine-average-wait-time](./sql-engine-average-wait-time.md) — SQL workers are spending too much time waiting on resources such as storage, locks, memory grants, CPU schedulers, or network I/O.
- [sql-engine-cpu-utilization](./sql-engine-cpu-utilization.md) — Schedulers are saturated or close to saturation.
- [sql-engine-stolen-server-memory](./sql-engine-stolen-server-memory.md) — Memory consumers such as query workspace, plan cache, locks, CLR, columnstore, or in-memory features are taking memory that could otherwise support data cache.
- [sql-engine-thread-count](./sql-engine-thread-count.md) — Worker starvation is possible.
- [sql-user-resource-pool-memory](./sql-user-resource-pool-memory.md) — Queries assigned to the pool may experience memory grant waits, spills, or failures.

### Security

- [sql-database-securables-config](./sql-database-securables-config.md) — The database is online but permissions prevent reliable per-database monitoring.
- [sql-engine-securables-config](./sql-engine-securables-config.md) — The probe is not seeing enough metadata to reliably monitor SQL.
- [sql-error-33108-tde-key-failure](./sql-error-33108-tde-key-failure.md) — SQL cannot access TDE key material required to decrypt or protect the database encryption key.
- [sql-error-33111-tde-cert-failure](./sql-error-33111-tde-cert-failure.md) — SQL cannot access TDE key material required to decrypt or protect the database encryption key.

### Service Broker

- [sql-error-9689-sb-manager-shutdown](./sql-error-9689-sb-manager-shutdown.md) — Service Broker infrastructure is not running correctly.
- [sql-error-9694-sb-cannot-start](./sql-error-9694-sb-cannot-start.md) — Service Broker infrastructure is not running correctly.
- [sql-error-9695-sb-task-mgr-memory](./sql-error-9695-sb-task-mgr-memory.md) — Service Broker infrastructure is not running correctly.
- [sql-error-9696-sb-event-handler-cannot-start](./sql-error-9696-sb-event-handler-cannot-start.md) — Service Broker infrastructure is not running correctly.
- [sql-error-9697-sb-cannot-start-on-database](./sql-error-9697-sb-cannot-start-on-database.md) — Service Broker infrastructure is not running correctly.
- [sql-error-9698-sb-security-mgr-cannot-start](./sql-error-9698-sb-security-mgr-cannot-start.md) — Service Broker infrastructure is not running correctly.

### Storage

- [sql-database-rows-size-percent](./sql-database-rows-size-percent.md) — Data files or their hosting volumes are running out of usable space.
- [sql-database-tx-log-space](./sql-database-tx-log-space.md) — The log cannot truncate or grow enough for current workload.
- [sql-engine-disk-latency-read](./sql-engine-disk-latency-read.md) — Data and log reads are taking too long at the Windows storage layer.
- [sql-engine-disk-latency-write](./sql-engine-disk-latency-write.md) — Log and data writes are slow.
- [sql-error-1101-allocate-failed-tempdb](./sql-error-1101-allocate-failed-tempdb.md) — SQL could not allocate database pages.
- [sql-error-1105-disk-full](./sql-error-1105-disk-full.md) — SQL could not allocate database pages.
- [sql-error-823-io-logical-error](./sql-error-823-io-logical-error.md) — SQL has evidence of unreliable storage or page-level corruption.
- [sql-error-824-io-logical-checksum](./sql-error-824-io-logical-checksum.md) — SQL has evidence of unreliable storage or page-level corruption.
- [sql-error-825-read-retry](./sql-error-825-read-retry.md) — SQL has evidence of unreliable storage or page-level corruption.
- [sql-error-832-constant-page-failure](./sql-error-832-constant-page-failure.md) — SQL has evidence of unreliable storage or page-level corruption.
- [sql-error-9002-log-full](./sql-error-9002-log-full.md) — The transaction log cannot reuse or grow enough space.

## Source notes

- UnitMonitor-derived runbooks follow `docs/research/sql/04-azmon-feasibility.md` per-monitor mapping plus `raw/agent-b-unit-monitors.md` monitor IDs.
- Windows Event Log runbooks follow the 10 DCR-Events mapping rows in the feasibility design, including the SQL Agent job failure rule called out for v1 documentation.
- ERRORLOG runbooks follow the named high-value error-number buckets in the feasibility design and user inventory. The inventory contains 29 explicit ERRORLOG alert IDs; this KB contains 78 alert runbooks plus this index.

# SQL Server — SQL error 33108 TDE key failure

**Alert ID:** `sql-error-33108-tde-key-failure`
**Severity:** 1 Critical
**Category:** Security
**Detection mechanism:** Custom Text Logs (ERRORLOG) DCR
**Source SCOM monitor / rule (if any):** `EventRule.DBEngine.TDE_key_failure_33108_Rule`

---

## Symptom

The proposed alert fires when SQL ERRORLOG ingestion sees error 33108. Typical message text: "Cannot obtain the database encryption key or server certificate required by Transparent Data Encryption."

## What it means

SQL cannot access TDE key material required to decrypt or protect the database encryption key. Affected encrypted databases may fail to open after restart or restore.

## Common causes

- Certificate/asymmetric key missing after restore or migration
- Master key not openable or service master key changed
- Certificate private key not restored
- Permission denied to key/certificate metadata
- TDE protector expired or accidentally dropped

## How to diagnose

Start with the 24-hour Log Analytics query to identify the affected computer, instance, database, replica, event, and payload. Then connect to the affected SQL Server instance with a DBA account and run the T-SQL checks shown below.

Record these triage facts before changing anything:

- Alert fire time, first-seen time, and whether this is recurring.
- Affected computer, SQL instance, database, AG, replica, file, job, or error number from the payload.
- Current SQL Server build from `SELECT @@VERSION` and any recent CU, OS, storage, or failover change.
- Whether the condition aligns with an approved maintenance window, failover test, restore, or deployment.
- Whether the same host also has storage, cluster, service, or Agent alerts in the last 24 hours.
- Whether user impact is active: failed connections, failed writes, stale secondary reads, missed backups, or job backlog.

```kusto
SqlErrorLog_CL
| where TimeGenerated > ago(24h)
| extend ErrorNumber = toint(coalesce(ErrorNumber, ErrorNumber_s, extract(@"Error:\s*(\d+)", 1, RawData)))
| where ErrorNumber == 33108
| extend InstanceName = tostring(coalesce(InstanceName_s, InstanceName))
| extend DatabaseName = tostring(coalesce(DatabaseName_s, DatabaseName))
| project TimeGenerated, Computer, InstanceName, DatabaseName, ErrorNumber,
          Severity=tostring(coalesce(Severity_s, Severity)), Message=tostring(coalesce(Message, RawData))
| order by TimeGenerated desc
```

```sql
-- T-SQL the DBA runs on the affected instance
SELECT DB_NAME(dek.database_id) AS database_name, dek.encryption_state,
       dek.encryption_state_desc, dek.percent_complete, dek.encryptor_type, dek.encryptor_thumbprint
FROM sys.dm_database_encryption_keys dek;

SELECT name, certificate_id, pvt_key_encryption_type_desc, expiry_date, thumbprint
FROM sys.certificates
ORDER BY expiry_date;
```

## How to fix

1. Do not restart or detach affected encrypted databases until key material is verified.
2. Restore missing certificate/private key from secure backup: `CREATE CERTIFICATE ... FROM FILE ... WITH PRIVATE KEY ...`.
3. Open/restore the database master key if needed and verify encryptor_thumbprint matches a certificate.
4. Back up the certificate and private key immediately after repair and store it outside the SQL host.
5. Success is TDE DMV state healthy and encrypted database opens after a controlled failover/restart test.

6. Confirm the next probe sample or alert evaluation returns healthy before closing the incident.
7. Capture the final root cause, corrective action, and any threshold or runbook tuning needed for future alerts.

## Escalation

Escalate to DBA/security key owners if certificate/private-key backups are missing, the service master key changed, or encrypted databases cannot open after restore. Do not drop or regenerate TDE protectors during triage.

## Related alerts

- [sql-engine-securables-config](./sql-engine-securables-config.md)
- [sql-database-securables-config](./sql-database-securables-config.md)
- [sql-error-33111-tde-cert-failure](./sql-error-33111-tde-cert-failure.md)

## References

- Feasibility design: `docs/research/sql/04-azmon-feasibility.md`
- Source mapping: `docs/research/sql/02-alerts-and-detection.md`
- Raw research: `docs/research/sql/raw/agent-c-rules-part1.md` / `agent-c-rules-part2.md` (for rule IDs)
- Microsoft docs: https://learn.microsoft.com/sql/relational-databases/errors-events/mssqlserver-33108-database-engine-error
- Kevin Holman: https://kevinholman.com (search for the SCOM monitor name when relevant)

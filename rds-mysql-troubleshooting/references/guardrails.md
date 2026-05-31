# RDS MySQL Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any RDS MySQL issue.

## Guardrail 1: No SUPER Privilege on RDS
RDS MySQL does not grant the SUPER privilege. Do not suggest `SET GLOBAL` for restricted system variables, `CHANGE REPLICATION SOURCE TO`, or `KILL` directly. Use RDS-provided stored procedures: `CALL mysql.rds_kill(<id>)`, `CALL mysql.rds_set_configuration('binlog retention hours', 24)`.

## Guardrail 2: InnoDB Only for Production
MyISAM is not crash-safe and does not replicate reliably on RDS. Do not suggest creating MyISAM tables. All production tables should use InnoDB. The `default_storage_engine` parameter should remain InnoDB.

## Guardrail 3: Multi-AZ Standby Is Not Readable
The Multi-AZ standby instance is not accessible for reads. Do not suggest connecting to or querying the standby. For read scaling, use read replicas. Multi-AZ is for high availability only.

## Guardrail 4: PITR Creates a New Instance
Point-in-time recovery restores to a NEW instance with a new endpoint. Do not suggest in-place PITR. Applications must be updated to point to the new endpoint after restore.

## Guardrail 5: Read Replica Lag Is Expected
MySQL read replicas use asynchronous replication. Some lag is normal under write-heavy workloads. Do not diagnose normal async lag as a bug. Investigate only when lag is growing unbounded or replication is broken (Seconds_Behind_Master = NULL).

## Guardrail 6: Binary Log Format Must Be ROW
For replication to work correctly, `binlog_format` must be ROW. Do not suggest STATEMENT or MIXED format as they can cause replication inconsistencies. ROW-based replication is required for read replicas.

## Guardrail 7: max_connections Is Memory-Bound
The default `max_connections` is `{DBInstanceClassMemory/12582880}`. Each connection consumes memory. Setting max_connections too high causes OOM kills and instance instability. Do not blindly increase max_connections — scale the instance class or use RDS Proxy for connection pooling.

## Guardrail 8: No Direct File System Access
There is no SSH or OS-level access to RDS MySQL. Do not suggest editing `my.cnf`, accessing data directories, or running OS commands. Use parameter groups for configuration and the RDS logs API for log access.

## Guardrail 9: Storage Autoscaling Has Cooldown
Storage autoscaling triggers when free space < 10% for > 5 minutes. There is a 6-hour cooldown between scaling events. Storage can only increase, never decrease. Do not suggest reducing allocated storage.

## Guardrail 10: RDS Proxy Requires IAM or Secrets Manager Auth
RDS Proxy authenticates using IAM authentication or Secrets Manager secrets. Do not suggest native MySQL auth directly to the proxy. The proxy handles the backend connection pooling transparently.

## Guardrail 11: Major Version Upgrades Require Pre-checks
Upgrading from MySQL 5.7 to 8.0 requires pre-upgrade compatibility checks. Deprecated features (e.g., `utf8mb3` default, `mysql_native_password` default) may break applications. Always run pre-upgrade validation before scheduling the upgrade.

## Guardrail 12: Encryption at Rest Is Immutable After Creation
Encryption at rest (KMS) must be enabled at instance creation. You cannot encrypt an existing unencrypted instance in-place. To encrypt, create an encrypted snapshot copy and restore from it. Do not suggest enabling encryption on a running unencrypted instance.

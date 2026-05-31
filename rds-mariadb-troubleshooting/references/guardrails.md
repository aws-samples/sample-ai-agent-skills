# RDS MariaDB Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any RDS MariaDB issue.

## Guardrail 1: No SUPER Privilege on RDS
RDS MariaDB does not grant SUPER. Do not suggest `SET GLOBAL` for restricted variables or `CHANGE MASTER TO` directly. Use RDS stored procedures: `CALL mysql.rds_kill()`, `CALL mysql.rds_set_configuration()`.

## Guardrail 2: MariaDB GTID ≠ MySQL GTID
MariaDB uses its own GTID format (domain-server_id-sequence). Do not apply MySQL GTID troubleshooting (UUID-based) to MariaDB. The replication protocols are different.

## Guardrail 3: Multi-AZ Standby Is Not Readable
The Multi-AZ standby is for failover only and cannot be queried. For read scaling, use read replicas. Do not suggest connecting to the standby.

## Guardrail 4: PITR Creates a New Instance
Point-in-time recovery restores to a NEW instance with a new endpoint. Do not suggest in-place PITR.

## Guardrail 5: InnoDB Is the Recommended Production Engine
While MariaDB supports Aria (crash-safe MyISAM replacement), InnoDB remains the recommended engine for production tables. Aria is used internally for temporary tables. Do not suggest Aria for production data.

## Guardrail 6: Thread Pool Is Enabled by Default
MariaDB on RDS has the thread pool enabled by default. This differs from MySQL on RDS. Do not suggest enabling it — it's already active. Tune `thread_pool_size` via parameter group if needed.

## Guardrail 7: max_connections Is Memory-Bound
Same as MySQL: each connection uses memory. Default is `{DBInstanceClassMemory/12582880}`. Do not blindly increase. Use connection pooling or scale the instance.

## Guardrail 8: No Direct File System Access
No SSH or OS access. Do not suggest editing `my.cnf` or accessing data directories. Use parameter groups and RDS logs API.

## Guardrail 9: Storage Can Only Increase
Storage autoscaling only grows storage, never shrinks. Plan `max-allocated-storage` carefully. 6-hour cooldown between scaling events.

## Guardrail 10: Binary Log Format Must Be ROW
For replication, `binlog_format` must be ROW. MIXED or STATEMENT can cause inconsistencies. ROW is required for read replicas.

## Guardrail 11: Encryption at Rest Is Immutable
Encryption must be enabled at creation. Cannot encrypt existing unencrypted instance in-place. Use snapshot copy with encryption.

## Guardrail 12: MariaDB Version Numbering
MariaDB uses 10.x versioning (10.4, 10.5, 10.6, 10.11, 11.4). Not all versions can upgrade directly. Check valid upgrade targets before scheduling upgrades. MariaDB 10.x is NOT MySQL 10.x.

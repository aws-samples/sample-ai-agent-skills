# RDS PostgreSQL Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any RDS PostgreSQL issue.

## Guardrail 1: No OS-Level Access
There is no SSH or filesystem access to RDS PostgreSQL. Do not suggest editing `postgresql.conf`, `pg_hba.conf`, or running OS commands. Use parameter groups for configuration and the RDS logs API for log access.

## Guardrail 2: rds_superuser Is Not PostgreSQL superuser
The `rds_superuser` role has elevated but restricted privileges. Some superuser-only functions (e.g., `pg_read_file`, `pg_ls_dir`, direct WAL manipulation) are not available. Do not suggest operations requiring true superuser access.

## Guardrail 3: Never Disable Autovacuum Globally
Disabling autovacuum leads to table bloat, index bloat, and transaction ID wraparound (which forces the database into read-only mode). Do not suggest `autovacuum=off`. Per-table tuning is acceptable for specific workloads.

## Guardrail 4: Transaction ID Wraparound Is an Emergency
If `age(datfrozenxid)` approaches 2 billion, the database will enter read-only mode to prevent data loss. This is a critical emergency. Do not dismiss wraparound warnings. Ensure autovacuum is running and completing on large tables.

## Guardrail 5: PITR Creates a New Instance
Point-in-time recovery restores to a NEW instance with a new endpoint. Do not suggest in-place PITR. Applications must be updated to point to the new endpoint.

## Guardrail 6: shared_buffers Should Not Exceed 40% of Memory
PostgreSQL relies on the OS page cache for additional caching. Setting `shared_buffers` too high (> 40% of instance memory) reduces OS cache effectiveness and can degrade performance. The RDS default of ~25% is usually optimal.

## Guardrail 7: Logical Replication Requires Explicit Configuration
Logical replication requires `rds.logical_replication=1` (which sets `wal_level=logical`). This increases WAL generation. Replication slots that are not consumed will cause WAL to accumulate and fill storage. Monitor `pg_replication_slots` for inactive slots.

## Guardrail 8: Extensions Must Be RDS-Allowlisted
Not all PostgreSQL extensions are available on RDS. Use `SHOW rds.allowed_extensions` to check. Some extensions (e.g., `pg_cron`, `pgaudit`) require parameter group changes. Do not suggest installing extensions not on the allowlist.

## Guardrail 9: max_connections Is Memory-Bound
Each PostgreSQL connection uses ~10MB of memory. The default formula is `LEAST({DBInstanceClassMemory/9531392}, 5000)`. Setting max_connections too high causes OOM. Use RDS Proxy or PgBouncer for connection pooling instead of increasing max_connections.

## Guardrail 10: Major Version Upgrades Use pg_upgrade
Major version upgrades (e.g., 14→16) use `pg_upgrade` internally. This requires downtime. Extensions must be compatible with the target version. Pre-upgrade checks are essential. Some extensions may need to be dropped and recreated.

## Guardrail 11: WAL Disk Usage Can Fill Storage
`TransactionLogsDiskUsage` CloudWatch metric tracks WAL size. Long-running transactions, inactive replication slots, and high write volume can cause WAL to grow and fill storage. Monitor this metric and set appropriate `max_slot_wal_keep_size`.

## Guardrail 12: Encryption at Rest Is Immutable After Creation
Encryption at rest (KMS) must be enabled at instance creation. You cannot encrypt an existing unencrypted instance in-place. To encrypt, create an encrypted snapshot copy and restore from it.

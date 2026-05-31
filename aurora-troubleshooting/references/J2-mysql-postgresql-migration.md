---
title: "J2 — MySQL/PostgreSQL Migration to Aurora"
description: "Diagnose native MySQL or PostgreSQL migration to Aurora issues"
status: active
severity: HIGH
triggers:
  - "migration to Aurora"
  - "mysqldump"
  - "pg_dump"
  - "snapshot migration"
  - "binlog replication migration"
  - "logical replication migration"
owner: devops-agent
objective: "Identify and resolve issues when migrating from MySQL/PostgreSQL to Aurora"
context: "Migration to Aurora can use several methods: RDS snapshot migration (from RDS MySQL/PostgreSQL), mysqldump/pg_dump logical migration, binlog replication (MySQL), logical replication (PostgreSQL), S3 import (Aurora MySQL), or DMS. Each method has different trade-offs for downtime, complexity, and data consistency."
---

## Phase 1 — Triage

MUST:
- Identify migration method being used
- For snapshot migration — check snapshot compatibility:
  ```
  aws rds describe-db-cluster-snapshots --db-cluster-snapshot-identifier <snapshot-id>
  ```
- For S3 import (Aurora MySQL) — check import status:
  ```
  aws rds describe-events --source-identifier <cluster-id> --source-type db-cluster --duration 1440
  ```
- For binlog replication migration — check replication status:
  ```sql
  -- On Aurora MySQL target
  SHOW SLAVE STATUS\G
  ```
- For logical replication migration (PostgreSQL) — check subscription:
  ```sql
  -- On Aurora PostgreSQL target
  SELECT * FROM pg_stat_subscription;
  ```

SHOULD:
- Verify engine version compatibility between source and Aurora target
- Check for unsupported features or data types in the target engine
- For Aurora MySQL — verify character set and collation compatibility:
  ```sql
  SHOW VARIABLES LIKE 'character_set%';
  SHOW VARIABLES LIKE 'collation%';
  ```
- For Aurora PostgreSQL — verify extension compatibility:
  ```sql
  SELECT * FROM pg_available_extensions WHERE installed_version IS NOT NULL;
  ```

MAY:
- Run pre-migration assessment (DMS or manual compatibility check)
- Check for stored procedures, triggers, or views that may need modification

## Phase 2 — Remediate

MUST:
- For snapshot migration failures: verify source snapshot is compatible with Aurora engine version
- For replication-based migration: ensure source binlog/WAL retention is sufficient
- For logical dump/restore: verify dump file is compatible with Aurora engine

SHOULD:
- Use Aurora MySQL S3 import for large MySQL databases (faster than mysqldump):
  ```
  aws rds restore-db-cluster-from-s3 --db-cluster-identifier <cluster-id> \
    --engine aurora-mysql --s3-bucket-name <bucket> --s3-prefix <prefix> \
    --s3-ingestion-role-arn <role-arn> --source-engine mysql --source-engine-version <version>
  ```
- For minimal downtime: use replication-based migration (binlog or logical replication) with cutover
- Test migration on a non-production environment first

MAY:
- Use Aurora blue/green deployments for in-place engine upgrades
- Implement application-level dual-write during migration for zero-downtime cutover

## Common Issues

- symptoms: "Snapshot restore to Aurora failed — incompatible engine version"
  diagnosis: "Source RDS snapshot engine version not compatible with Aurora."
  resolution: "Upgrade source to a compatible version first, or use DMS/logical migration."

- symptoms: "S3 import failed with permission error"
  diagnosis: "IAM role for S3 import missing permissions."
  resolution: "Grant the S3 import role s3:GetObject, s3:ListBucket on the source bucket."

- symptoms: "Replication-based migration lag too high for cutover"
  diagnosis: "Source write rate exceeds replication throughput."
  resolution: "Scale up Aurora target. Reduce source write rate during cutover window."

## Safety Ratings
- GREEN: describe-db-clusters, describe-db-cluster-snapshots, SHOW SLAVE STATUS, pg_stat_subscription, SHOW VARIABLES — read-only inspection
- YELLOW: restore-db-cluster-from-s3, modify-db-cluster — recoverable migration operations
- RED: delete-db-cluster, dropping source database — destructive operations, potential data loss

## Escalation Conditions
- "Database serves production traffic"
- "Migration failure affecting data consistency between source and target"
- "Fix requires modifying source or target database configuration"
- "Replication lag too high for planned cutover window"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, S3 bucket ARNs, IAM role ARNs
- HIGH: migration data (contains full database content), replication user credentials
- MEDIUM: engine version compatibility, character set/collation settings, extension lists

## Prohibited Actions
- "NEVER suggest deleting the source database before confirming migration is complete and validated"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest cutting over without validating data consistency"
- "NEVER suggest modifying production parameter groups without testing"

## Phase 3 — Rollback
- "If migration fails, revert application to source database"
- "Restore from snapshot if Aurora target has data issues"
- "If replication-based migration has errors, rebuild replication from a fresh snapshot"
- "Keep source database running until migration is fully validated"

## Output Format

```yaml
root_cause: "migration_issue — <specific_cause>"
evidence:
  - type: migration_method
    content: "<method being used>"
  - type: migration_status
    content: "<current migration status>"
  - type: compatibility
    content: "<engine version and feature compatibility>"
severity: HIGH
mitigation:
  immediate: "Fix migration error and resume"
  long_term: "Document migration runbook and test in non-production"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling encryption"
  - "NEVER force failover without understanding impact"

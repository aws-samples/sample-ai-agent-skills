---
title: "H1 — Cassandra Migration"
description: "Diagnose issues migrating from Apache Cassandra to Amazon Keyspaces"
status: active
severity: HIGH
triggers:
  - "Cassandra migration"
  - "migrate to Keyspaces"
  - "migration errors"
  - "schema migration"
  - "data migration"
owner: devops-agent
objective: "Resolve issues encountered during Cassandra to Keyspaces migration"
context: "Migrating from Apache Cassandra to Keyspaces requires schema adaptation (removing unsupported features), data migration (CQL COPY, AWS DMS, CQLReplicator, or AWS Glue), and application changes (connection strings, authentication, CQL compatibility). Schema must be modified to remove materialized views, UDFs, UDAs, triggers, COMPACT STORAGE, and unsupported options."
---

## Phase 1 — Triage

MUST:
- Identify the migration phase (schema, data, application cutover)
- Review Cassandra schema for unsupported features: materialized views, UDFs, UDAs, triggers, COMPACT STORAGE
- Check CQL compatibility of all application queries
- Verify Keyspaces service limits vs current Cassandra usage

SHOULD:
- Assess data volume and estimate migration time
- Check partition key design for Keyspaces compatibility
- Verify all data types are supported in Keyspaces
- Test application connectivity with Keyspaces endpoint

MAY:
- Run a parallel write test to both Cassandra and Keyspaces
- Benchmark query performance on Keyspaces vs Cassandra

## Phase 2 — Remediate

MUST:
- Adapt schema: remove unsupported features, adjust data types
- Choose migration tool: CQL COPY (small), AWS DMS (medium), CQLReplicator (live), AWS Glue (large)
- Update application connection strings to Keyspaces endpoint (port 9142, TLS)
- Configure authentication (SigV4 or service-specific credentials)

SHOULD:
- Migrate in phases: schema first, then data, then application cutover
- Run validation queries to verify data integrity after migration
- Implement dual-write during cutover for zero-downtime migration

MAY:
- Use AWS SCT (Schema Conversion Tool) for schema analysis
- Implement rollback plan to Cassandra if issues arise

## Common Issues

- symptoms: "Schema creation fails due to unsupported features"
  diagnosis: "Cassandra schema uses features not available in Keyspaces."
  resolution: "Remove materialized views, UDFs, UDAs, triggers, COMPACT STORAGE. Adapt data types."

- symptoms: "CQL COPY fails on large dataset"
  diagnosis: "CQL COPY is single-threaded and not suitable for large datasets."
  resolution: "Use AWS DMS, CQLReplicator, or AWS Glue for large-scale data migration."

- symptoms: "Application errors after migration"
  diagnosis: "CQL queries use unsupported syntax or features."
  resolution: "Audit all CQL queries for Keyspaces compatibility. See C1-cql-compatibility runbook."

## Output Format

```yaml
root_cause: "cassandra_migration — <specific_cause>"
evidence:
  - type: migration_phase
    content: "<schema, data, or application cutover>"
  - type: compatibility_issues
    content: "<unsupported features identified>"
  - type: migration_tool
    content: "<tool used and its status>"
severity: HIGH
mitigation:
  immediate: "Fix specific migration blocker"
  long_term: "Complete migration with validation and monitoring"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never drop keyspaces or tables containing production data
- Never modify capacity mode on tables during peak traffic without load testing
- Never alter TTL settings without understanding data retention requirements
- Never change encryption settings on tables with active workloads
- Never disable point-in-time recovery on production tables

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore tables from point-in-time recovery if data is accidentally modified
- Revert capacity mode changes if throttling increases after modification
- Re-enable previous auto-scaling configurations if performance degrades

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"

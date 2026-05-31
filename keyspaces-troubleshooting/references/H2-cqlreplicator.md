---
title: "H2 — CQLReplicator"
description: "Diagnose Amazon Keyspaces CQLReplicator issues for live migration"
status: active
severity: MEDIUM
triggers:
  - "CQLReplicator"
  - "live migration"
  - "replication lag"
  - "replicator errors"
  - "continuous migration"
owner: devops-agent
objective: "Resolve CQLReplicator configuration and operation issues"
context: "CQLReplicator is an open-source tool for live data migration from Apache Cassandra to Amazon Keyspaces. It reads from Cassandra and writes to Keyspaces in near-real-time. It requires Java, connectivity to both clusters, and appropriate credentials. Common issues include connectivity, throughput limits, schema mismatches, and replication lag."
---

## Phase 1 — Triage

MUST:
- Verify CQLReplicator is running and check its logs
- Check connectivity to both source Cassandra and target Keyspaces
- Verify Keyspaces write capacity is sufficient for replication throughput
- Check for write throttling on the target table: `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name WriteThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check replication lag metrics
- Verify schema compatibility between source and target tables
- Check CQLReplicator configuration for correct endpoints and credentials
- Monitor consumed write capacity during replication

MAY:
- Compare row counts between source and target
- Check for data type conversion issues

## Phase 2 — Remediate

MUST:
- Fix connectivity issues to source or target
- Increase Keyspaces write capacity if throttling during replication
- Ensure schema matches between source Cassandra and target Keyspaces table
- Restart CQLReplicator after configuration changes

SHOULD:
- Tune CQLReplicator batch size and parallelism for optimal throughput
- Monitor replication progress and lag
- Validate data integrity periodically during migration

MAY:
- Use multiple CQLReplicator instances for parallel migration
- Implement checkpointing for resumable migration

## Common Issues

- symptoms: "CQLReplicator cannot connect to Keyspaces"
  diagnosis: "Missing TLS configuration or incorrect endpoint/port."
  resolution: "Configure TLS with Starfield certificate. Use port 9142 and correct regional endpoint."

- symptoms: "Replication falling behind — high lag"
  diagnosis: "Write throughput to Keyspaces is insufficient for the replication rate."
  resolution: "Increase Keyspaces write capacity. Tune CQLReplicator parallelism. Use on-demand mode."

- symptoms: "Data type mismatch errors during replication"
  diagnosis: "Source Cassandra uses data types not supported in Keyspaces."
  resolution: "Modify target schema to use compatible types. Implement data transformation in replication."

## Output Format

```yaml
root_cause: "cqlreplicator — <specific_cause>"
evidence:
  - type: replicator_status
    content: "<CQLReplicator running status and logs>"
  - type: connectivity
    content: "<source and target connectivity status>"
  - type: throughput
    content: "<replication rate vs write capacity>"
severity: MEDIUM
mitigation:
  immediate: "Fix connectivity or capacity issues"
  long_term: "Monitor replication and validate data integrity"
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

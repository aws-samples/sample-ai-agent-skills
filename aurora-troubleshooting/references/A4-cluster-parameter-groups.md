---
title: "A4 — Aurora Cluster Parameter Groups"
description: "Diagnose Aurora cluster and instance parameter group issues"
status: active
severity: MEDIUM
triggers:
  - "parameter group"
  - "cluster parameter"
  - "instance parameter"
  - "parameter mismatch"
  - "pending reboot"
  - "parameter not taking effect"
owner: devops-agent
objective: "Identify and resolve Aurora parameter group configuration issues"
context: "Aurora has two levels of parameter groups: cluster parameter groups (apply to all instances, engine-level settings) and instance parameter groups (apply to individual instances). Some parameters require a reboot to take effect. Parameter group family must match the engine version."
---

## Phase 1 — Triage

MUST:
- Check cluster parameter group:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].DBClusterParameterGroup'
  ```
- Check instance parameter groups:
  ```
  aws rds describe-db-instances --filters Name=db-cluster-id,Values=<cluster-id> \
    --query 'DBInstances[].{Id:DBInstanceIdentifier,ParamGroup:DBParameterGroups}'
  ```
- List cluster parameters with pending changes:
  ```
  aws rds describe-db-cluster-parameters --db-cluster-parameter-group-name <cluster-param-group> \
    --query 'Parameters[?ApplyMethod==`pending-reboot`]'
  ```
- Check for pending reboot status:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].DBClusterMembers[].{Id:DBInstanceIdentifier,PendingReboot:IsClusterWriter}'
  ```

SHOULD:
- Compare current vs default parameter values:
  ```
  aws rds describe-db-cluster-parameters --db-cluster-parameter-group-name <cluster-param-group> \
    --source user
  ```
- Check instance-level parameter overrides:
  ```
  aws rds describe-db-parameters --db-parameter-group-name <instance-param-group> --source user
  ```
- For Aurora MySQL — verify key parameters:
  ```sql
  SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
  SHOW VARIABLES LIKE 'max_connections';
  SHOW VARIABLES LIKE 'binlog_format';
  ```
- For Aurora PostgreSQL — verify key parameters:
  ```sql
  SHOW shared_buffers;
  SHOW max_connections;
  SHOW rds.logical_replication;
  ```

MAY:
- Check if parameter group family matches engine version
- Review parameter group change history via CloudTrail

## Phase 2 — Remediate

MUST:
- For pending-reboot parameters: reboot the affected instances (or entire cluster for cluster params)
- For parameter group family mismatch: create a new parameter group with the correct family
- For cluster vs instance parameter confusion: identify which level the parameter belongs to

SHOULD:
- Apply dynamic parameters first (no reboot needed), then batch static parameter changes with a planned reboot
- Document all parameter changes and their rationale
- Test parameter changes on a clone before applying to production

MAY:
- Use Aurora blue/green deployments for major parameter changes
- Create custom parameter groups rather than modifying the default group

## Common Issues

- symptoms: "Parameter change not taking effect"
  diagnosis: "Static parameter requires reboot, or parameter was set at wrong level (cluster vs instance)."
  resolution: "Check ApplyMethod. Reboot if pending-reboot. Verify parameter is set at correct level."

- symptoms: "Parameter group family mismatch after engine upgrade"
  diagnosis: "Parameter group family (e.g., aurora-mysql5.7) doesn't match new engine version (8.0)."
  resolution: "Create a new parameter group with the correct family and associate it."

- symptoms: "binlog_format or rds.logical_replication not taking effect"
  diagnosis: "These are cluster-level parameters. Must be set in the cluster parameter group."
  resolution: "Modify the cluster parameter group and reboot all instances."

## Safety Ratings
- GREEN: describe-db-cluster-parameters, describe-db-parameters, describe-db-clusters, SHOW VARIABLES/SHOW — read-only parameter inspection
- YELLOW: modify-db-cluster-parameter-group, modify-db-parameter-group, reboot-db-instance — recoverable but may require reboot
- RED: reset-db-cluster-parameter-group, reset-db-parameter-group — resets all parameters to defaults, potentially destructive

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires parameter group change that needs reboot"
- "Fix requires failover of Aurora cluster"
- "Fix involves modifying encryption settings"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, parameter group values (may contain passwords or security-sensitive settings)
- HIGH: binlog_format, rds.logical_replication settings (affect replication security)
- MEDIUM: performance parameters, buffer pool sizes, connection limits

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix parameter issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest modifying production parameter groups without testing on a clone first"
- "NEVER suggest resetting parameter groups to defaults in production without review"
- "NEVER suggest force-failover in production without confirming application readiness"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"
- "Document previous parameter values before making changes to enable rollback"

## Output Format

```yaml
root_cause: "parameter_group_issue — <specific_cause>"
evidence:
  - type: parameter_config
    content: "<parameter group settings>"
  - type: pending_changes
    content: "<pending reboot parameters>"
severity: MEDIUM
mitigation:
  immediate: "Apply correct parameter at correct level and reboot if needed"
  long_term: "Maintain documented parameter group configurations in IaC"
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

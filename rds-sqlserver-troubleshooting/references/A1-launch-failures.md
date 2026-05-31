---
title: "A1 — RDS SQL Server Launch Failures"
description: "Diagnose failures when creating or launching an RDS SQL Server instance"
status: active
severity: HIGH
triggers:
  - "launch failed"
  - "create db instance failed"
  - "InsufficientDBInstanceCapacity"
  - "InvalidParameterValue"
  - "cannot create"
owner: devops-agent
objective: "Identify why an RDS SQL Server instance failed to launch and resolve the issue"
context: "RDS SQL Server launch failures commonly stem from edition/instance class incompatibilities, licensing issues, subnet group misconfiguration, parameter group family mismatch, insufficient capacity, or collation specification errors."
---

## Phase 1 — Triage

MUST:
- Check the RDS event log for the failed instance: `aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440`
- Verify the engine and version: `aws rds describe-db-engine-versions --engine sqlserver-ee --query 'DBEngineVersions[].EngineVersion'`
- Check instance class availability: `aws rds describe-orderable-db-instance-options --engine sqlserver-ee --engine-version <version> --query 'OrderableDBInstanceOptions[].DBInstanceClass'`
- Verify subnet group: `aws rds describe-db-subnet-groups --db-subnet-group-name <subnet-group>`
- Check parameter group family matches engine version: `aws rds describe-db-parameter-groups --db-parameter-group-name <param-group>`

SHOULD:
- Verify VPC security group exists: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check KMS key permissions if encrypted: `aws kms describe-key --key-id <key-id>`
- Verify option group engine compatibility: `aws rds describe-option-groups --option-group-name <option-group>`
- Check service quotas: `aws service-quotas get-service-quota --service-code rds --quota-code L-7B6409FD`
- Verify collation name is valid if specified

MAY:
- Check CloudTrail for the CreateDBInstance API call: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateDBInstance`
- Verify IAM permissions for the caller

## Phase 2 — Remediate

MUST:
- For edition/class mismatch: use `describe-orderable-db-instance-options` to find valid combinations
- For capacity errors: try a different AZ or instance class
- For parameter group mismatch: create a parameter group with the correct family (e.g., sqlserver-ee-15.0)
- For subnet group issues: ensure subnets span at least 2 AZs for Multi-AZ

SHOULD:
- Verify the SQL Server edition supports the requested features (Express vs Web vs Standard vs Enterprise)
- Confirm storage type and IOPS are valid for the instance class
- Verify collation name is a valid SQL Server collation before launch
- Test with a minimal configuration first, then add options

MAY:
- Open an AWS Support case if capacity is persistently unavailable

## Common Issues

- symptoms: "InvalidParameterCombination: The requested DB Instance class is not available for engine sqlserver-ex"
  diagnosis: "Instance class not supported for the SQL Server edition or version."
  resolution: "Use describe-orderable-db-instance-options to find valid combinations. Express has limited instance classes."

- symptoms: "InsufficientDBInstanceCapacity"
  diagnosis: "No capacity for the requested instance class in the selected AZ."
  resolution: "Try a different AZ or instance class."

- symptoms: "Invalid collation name specified"
  diagnosis: "Collation name is not a valid SQL Server collation."
  resolution: "Use a valid collation like SQL_Latin1_General_CP1_CI_AS. Check SQL Server documentation for valid collation names."

## Safety Ratings
- GREEN: describe-events, describe-db-engine-versions, describe-orderable-db-instance-options, describe-db-subnet-groups — read-only inspection
- YELLOW: modify-db-instance, modify-db-parameter-group — recoverable configuration changes
- RED: delete-db-instance, force-failover — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires parameter group change that needs reboot"
- "Fix requires failover of Multi-AZ instance"
- "Fix involves modifying encryption settings"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, KMS key IDs
- HIGH: CloudTrail events for CreateDBInstance (contain full parameter sets)
- MEDIUM: engine versions, instance class availability, collation settings

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix launch issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest force-failover in production without confirming application readiness"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "launch_failure — <specific_cause>"
evidence:
  - type: rds_event
    content: "<event message>"
  - type: engine_version
    content: "<describe-db-engine-versions output>"
severity: HIGH
mitigation:
  immediate: "Fix the specific launch parameter causing the failure"
  long_term: "Use infrastructure-as-code with validated parameter combinations"
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
  - "NEVER suggest making databases publicly accessible"
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting automated backups"

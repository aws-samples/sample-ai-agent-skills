---
title: "A1 — Aurora Cluster Creation Failures"
description: "Diagnose failures when creating an Aurora cluster"
status: active
severity: HIGH
triggers:
  - "create cluster failed"
  - "cluster creation failure"
  - "InsufficientDBClusterCapacity"
  - "InvalidParameterValue"
  - "cannot create aurora"
owner: devops-agent
objective: "Identify why an Aurora cluster failed to create and resolve the issue"
context: "Aurora cluster creation failures commonly stem from unsupported engine/instance class combinations, subnet group misconfiguration, parameter group family mismatch, insufficient capacity, VPC/security group issues, or KMS key permissions for encrypted clusters."
---

## Phase 1 — Triage

MUST:
- Check RDS events for the failed cluster: `aws rds describe-events --source-identifier <cluster-id> --source-type db-cluster --duration 1440`
- Verify engine and version availability: `aws rds describe-db-engine-versions --engine aurora-mysql --query 'DBEngineVersions[].EngineVersion'`
- Check instance class availability for the engine:
  ```
  aws rds describe-orderable-db-instance-options --engine aurora-mysql --engine-version <version> \
    --query 'OrderableDBInstanceOptions[].DBInstanceClass'
  ```
- Verify subnet group spans at least 2 AZs: `aws rds describe-db-subnet-groups --db-subnet-group-name <subnet-group>`
- Check cluster parameter group family matches engine: `aws rds describe-db-cluster-parameter-groups --db-cluster-parameter-group-name <cluster-param-group>`

SHOULD:
- Verify VPC security group exists: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check KMS key permissions if encrypted: `aws kms describe-key --key-id <key-id>`
- Verify IAM permissions for the caller: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateDBCluster`
- Check service quotas: `aws service-quotas get-service-quota --service-code rds --quota-code L-952B80B8`

MAY:
- Check if the engine version supports Serverless v2 if creating a Serverless cluster
- Verify Global Database compatibility if adding to a global cluster

## Phase 2 — Remediate

MUST:
- For capacity errors: try a different AZ or instance class
- For parameter group mismatch: create a cluster parameter group with the correct family (e.g., aurora-mysql8.0)
- For subnet group issues: ensure subnets span at least 2 AZs
- For KMS errors: grant the RDS service principal access to the KMS key

SHOULD:
- Verify the engine version supports requested features (Serverless v2, Global Database, backtrack)
- Confirm storage encryption settings are consistent (cannot mix encrypted and unencrypted in Global Database)
- Test with a minimal configuration first, then add features

MAY:
- Open an AWS Support case if capacity is persistently unavailable in the desired region

## Common Issues

- symptoms: "The requested DB instance class is not available for engine aurora-mysql"
  diagnosis: "Instance class not supported for the Aurora engine version."
  resolution: "Use describe-orderable-db-instance-options to find valid combinations."

- symptoms: "DB subnet group does not have subnets in enough Availability Zones"
  diagnosis: "Subnet group must span at least 2 AZs for Aurora."
  resolution: "Add subnets in additional AZs to the subnet group."

- symptoms: "The parameter group family aurora-mysql5.7 is not compatible with engine version 8.0"
  diagnosis: "Cluster parameter group family does not match the engine version."
  resolution: "Create a new cluster parameter group with the correct family."

## Safety Ratings
- GREEN: describe-db-clusters, describe-db-engine-versions, describe-orderable-db-instance-options, describe-db-subnet-groups, describe-events — read-only cluster inspection
- YELLOW: modify-db-cluster, modify-db-cluster-parameter-group — recoverable configuration changes
- RED: delete-db-cluster, force-failover — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires parameter group change that needs reboot"
- "Fix requires failover of Aurora cluster"
- "Fix involves modifying encryption settings"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, KMS key IDs (may expose encryption configuration)
- HIGH: CloudTrail events for CreateDBCluster (contain full parameter sets including security group IDs)
- MEDIUM: engine versions, instance class availability, subnet group configuration

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix creation issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest force-failover in production without confirming application readiness"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If cluster creation fails repeatedly, verify all parameters against describe-orderable-db-instance-options before retrying"

## Output Format

```yaml
root_cause: "cluster_creation_failure — <specific_cause>"
evidence:
  - type: rds_event
    content: "<event message>"
  - type: engine_version
    content: "<describe-db-engine-versions output>"
severity: HIGH
mitigation:
  immediate: "Fix the specific parameter causing the creation failure"
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
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling encryption"
  - "NEVER force failover without understanding impact"

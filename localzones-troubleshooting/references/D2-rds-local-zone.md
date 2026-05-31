---
title: "D2 — RDS Local Zone"
description: "Diagnose RDS deployment issues in Local Zones"
status: active
severity: HIGH
triggers: ["RDS Local Zone error", "cannot create RDS in Local Zone", "RDS engine not available", "RDS instance class not supported"]
owner: devops-agent
objective: "Resolve RDS deployment issues in Local Zones"
context: "RDS support in Local Zones is limited to specific DB engines (MySQL, PostgreSQL) and instance classes. Multi-AZ is NOT supported. DB subnet groups must include Local Zone subnets."
---

## Phase 1 — Triage

MUST:
- Check RDS engine availability: `aws rds describe-orderable-db-instance-options --engine <engine> --query 'OrderableDBInstanceOptions[?AvailabilityZones[?Name==`<local-zone-name>`]]|[*].{Engine:Engine,EngineVersion:EngineVersion,DBInstanceClass:DBInstanceClass}'`
- Verify DB subnet group includes Local Zone subnet: `aws rds describe-db-subnet-groups --db-subnet-group-name <name>`
- Check existing RDS instances: `aws rds describe-db-instances --query 'DBInstances[?AvailabilityZone==`<local-zone-name>`]'`

SHOULD:
- Verify the DB instance class is available in the Local Zone
- Check VPC security group allows database port access
- Verify storage type is supported

MAY:
- Review RDS event logs for creation errors
- Check parameter group compatibility

## Phase 2 — Remediate

MUST:
- Use a supported DB engine (MySQL or PostgreSQL) and version
- Select an available DB instance class for the Local Zone
- Create DB subnet group with Local Zone subnet included

SHOULD:
- Configure automated backups (snapshots stored in parent region)
- Set up read replica in parent region for disaster recovery

MAY:
- Configure Enhanced Monitoring for performance visibility
- Set up CloudWatch alarms for database metrics

## Safety Ratings

```
safety_ratings:
  - "describe-orderable-db-instance-options: GREEN — read-only availability check"
  - "describe-db-subnet-groups: GREEN — read-only subnet group details"
  - "create-db-instance: YELLOW — creates database with cost and data implications"
  - "delete-db-instance: RED — permanently deletes database and potentially data"
  - "modify-db-instance: YELLOW — changes database configuration, may cause downtime"
```

## Escalation Conditions

- Required DB engine not available in Local Zone requiring architecture change
- RDS instance class needed for performance not available in zone
- Multi-AZ requirement cannot be met in Local Zone
- Database migration to Local Zone requires extended downtime

## Data Sensitivity

- RDS instance configuration: MEDIUM — reveals database architecture
- DB subnet group: MEDIUM — reveals network configuration
- Database credentials: CRITICAL — must never be exposed
- RDS event logs: MEDIUM — may contain operational details

## Prohibited Actions

- NEVER delete an RDS instance without a final snapshot
- NEVER expose RDS endpoints publicly in Local Zones
- NEVER disable automated backups on production databases
- NEVER assume Multi-AZ works in Local Zones — it does not

## Phase 3 — Rollback

1. If RDS instance was created with wrong configuration, modify it: `aws rds modify-db-instance --db-instance-identifier <id> --apply-immediately`
2. If RDS instance must be deleted, create final snapshot first: `aws rds delete-db-instance --db-instance-identifier <id> --final-db-snapshot-identifier <snap-name>`
3. If DB subnet group was modified incorrectly, restore previous subnet configuration
4. Verify application connectivity after any RDS changes

## Common Issues

- symptoms: "CreateDBInstance returns InvalidParameterCombination for Local Zone"
  diagnosis: "DB engine, version, or instance class not supported in the Local Zone."
  resolution: "Check orderable options for the Local Zone. Use MySQL or PostgreSQL with supported instance classes."

- symptoms: "Cannot create Multi-AZ RDS in Local Zone"
  diagnosis: "Multi-AZ deployments are not supported in Local Zones."
  resolution: "Use single-AZ in Local Zone. Create read replica in parent region for DR."

## Output Format

```yaml
root_cause: "rds_local_zone — <specific_cause>"
evidence:
  - type: rds_error
    content: "<RDS configuration and availability details>"
severity: HIGH
mitigation:
  immediate: "Use supported engine and instance class"
  long_term: "Implement DR with parent region read replica"
```

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

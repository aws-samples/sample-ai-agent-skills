---
title: "A1 — Table Creation Failures"
description: "Diagnose Timestream database and table creation failures"
status: active
severity: HIGH
triggers:
  - "table creation failed"
  - "cannot create database"
  - "CreateTable error"
  - "CreateDatabase error"
owner: devops-agent
objective: "Identify why Timestream database or table creation is failing"
context: "Creation failures can be caused by IAM permissions, service quotas, invalid retention settings, KMS key issues, or naming conflicts. Databases and tables have separate creation requirements."
---

## Phase 1 — Triage

MUST:
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names timestream:CreateDatabase timestream:CreateTable`
- Verify database exists (for table creation): `aws timestream-write describe-database --database-name <database>`
- Check service quotas: `aws service-quotas get-service-quota --service-code timestream --quota-code L-4B58AFE3`
- Check CloudTrail for the creation failure: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateTable --max-results 10`

SHOULD:
- Verify the database/table name follows naming rules (3-256 chars, alphanumeric, hyphens, underscores, periods)
- Check if a database/table with the same name already exists
- Verify KMS key permissions if using customer-managed encryption

MAY:
- Check for SCP restrictions blocking Timestream operations
- Verify the region supports Timestream

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for timestream:CreateDatabase and timestream:CreateTable
- Create the database first if it doesn't exist:
  `aws timestream-write create-database --database-name <database>`
- Create the table with valid retention settings:
  `aws timestream-write create-table --database-name <database> --table-name <table> --retention-properties MemoryStoreRetentionPeriodInHours=24,MagneticStoreRetentionPeriodInDays=365`

SHOULD:
- Plan retention settings before creation
- Enable magnetic store writes if late-arriving data is expected

MAY:
- Configure KMS encryption for compliance
- Tag resources for cost allocation

## Common Issues

- symptoms: "ResourceNotFoundException when creating table"
  diagnosis: "Database does not exist."
  resolution: "Create the database first: aws timestream-write create-database."

- symptoms: "ConflictException — resource already exists"
  diagnosis: "Database or table with the same name already exists."
  resolution: "Use a different name or describe the existing resource."

- symptoms: "ValidationException — invalid retention"
  diagnosis: "Retention values outside valid range."
  resolution: "Memory: 1-8760 hours. Magnetic: 1-73000 days."

## Output Format

```yaml
root_cause: "creation_failure — <specific_cause>"
evidence:
  - type: iam_permissions
    content: "<IAM evaluation result>"
  - type: database_status
    content: "<database existence check>"
  - type: cloudtrail_event
    content: "<creation error from CloudTrail>"
severity: HIGH
mitigation:
  immediate: "Fix the specific creation blocker"
  long_term: "Automate Timestream resource creation with IaC"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Time-series data may contain IoT sensor readings and telemetry
- Query results may reveal operational patterns and metrics
- Scheduled query outputs may aggregate sensitive measurements
- Classification: CONFIDENTIAL - operational telemetry and measurement data

## Prohibited Actions

- NEVER delete Timestream tables containing production time-series data
- NEVER reduce memory store retention below the current data window
- NEVER modify magnetic store retention without data lifecycle review
- NEVER cancel scheduled queries that feed production dashboards

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"

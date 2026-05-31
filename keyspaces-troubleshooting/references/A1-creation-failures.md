---
title: "A1 — Table Creation Failures"
description: "Diagnose why an Amazon Keyspaces table creation is failing"
status: active
severity: HIGH
triggers:
  - "table creation failed"
  - "cannot create table"
  - "CREATE TABLE error"
  - "keyspace not found"
  - "schema validation error"
owner: devops-agent
objective: "Identify why table creation is failing and resolve the issue"
context: "Keyspaces table creation can fail due to invalid schema, unsupported CQL features, missing keyspace, IAM permission issues, or service limits. Tables are created asynchronously — status should be checked after creation. Keyspaces supports a subset of CQL data types and table options."
---

## Phase 1 — Triage

MUST:
- Verify keyspace exists: `aws keyspaces get-keyspace --keyspace-name <keyspace>`
- Check IAM permissions for the caller: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names cassandra:Create`
- Check CloudTrail for creation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateTable --max-results 10`
- Verify table does not already exist: `aws keyspaces list-tables --keyspace-name <keyspace>`

SHOULD:
- Check service quotas: `aws service-quotas get-service-quota --service-code cassandra --quota-code L-0B8E498B`
- Verify CQL syntax is Keyspaces-compatible (no unsupported types or options)
- Check for reserved keyword conflicts in column names

MAY:
- Review the full CQL CREATE TABLE statement for unsupported options
- Check if the table name contains invalid characters

## Phase 2 — Remediate

MUST:
- Create the keyspace first if it does not exist: `aws keyspaces create-keyspace --keyspace-name <keyspace>`
- Fix any unsupported CQL syntax (remove COMPACT STORAGE, custom compaction, etc.)
- Ensure IAM policy includes `cassandra:Create` permission

SHOULD:
- Use AWS CLI for table creation to get detailed error messages: `aws keyspaces create-table --keyspace-name <keyspace> --table-name <table> --schema-definition '<schema-json>'`
- Verify table status after creation: `aws keyspaces get-table --keyspace-name <keyspace> --table-name <table>`

MAY:
- Request service quota increase if at table limit
- Use tags for resource organization: `aws keyspaces tag-resource --resource-arn <table-arn> --tags key=Environment,value=prod`

## Common Issues

- symptoms: "Keyspace not found error during CREATE TABLE"
  diagnosis: "The target keyspace does not exist."
  resolution: "Create the keyspace first: aws keyspaces create-keyspace --keyspace-name <keyspace>"

- symptoms: "Unsupported feature error in CREATE TABLE"
  diagnosis: "CQL statement uses features not supported by Keyspaces (COMPACT STORAGE, custom compaction)."
  resolution: "Remove unsupported options. Use only Keyspaces-compatible CQL syntax."

- symptoms: "AccessDeniedException on table creation"
  diagnosis: "IAM policy missing cassandra:Create permission."
  resolution: "Add cassandra:Create permission to the IAM policy for the Keyspaces resource."

## Output Format

```yaml
root_cause: "table_creation_failure — <specific_cause>"
evidence:
  - type: keyspace_status
    content: "<keyspace existence check>"
  - type: iam_permissions
    content: "<IAM policy evaluation>"
  - type: cloudtrail
    content: "<creation event details>"
severity: HIGH
mitigation:
  immediate: "Fix schema or permissions and retry creation"
  long_term: "Validate CQL compatibility before deployment"
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

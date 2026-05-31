---
title: "A2 — Schema Issues"
description: "Diagnose Amazon Keyspaces schema modification and compatibility problems"
status: active
severity: MEDIUM
triggers:
  - "ALTER TABLE failed"
  - "schema change error"
  - "column type mismatch"
  - "cannot add column"
  - "schema modification"
owner: devops-agent
objective: "Identify and resolve schema-related issues in Amazon Keyspaces"
context: "Keyspaces supports a subset of CQL schema operations. ALTER TABLE can add columns but cannot drop columns or change column types. Schema changes are asynchronous. Some CQL data types have limitations or are unsupported."
---

## Phase 1 — Triage

MUST:
- Check current table schema: `aws keyspaces get-table --keyspace-name <keyspace> --table-name <table>`
- Verify the ALTER TABLE statement uses supported operations
- Check CloudTrail for schema change events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateTable --max-results 10`
- Verify table status is ACTIVE before schema changes: `aws keyspaces get-table --keyspace-name <keyspace> --table-name <table> --query 'status'`

SHOULD:
- Review supported CQL data types for Keyspaces
- Check if column names conflict with CQL reserved words
- Verify no concurrent schema modifications are in progress

MAY:
- Compare schema with the original Cassandra schema if migrating
- Check application compatibility with the current schema

## Phase 2 — Remediate

MUST:
- Wait for table status to be ACTIVE before retrying schema changes
- Use only supported ALTER TABLE operations (ADD column)
- For column type changes, create a new table with the correct schema and migrate data

SHOULD:
- Test schema changes in a non-production keyspace first
- Use AWS CLI for schema changes to get detailed error messages

MAY:
- Document schema differences between Cassandra and Keyspaces for migration
- Implement schema versioning in application code

## Common Issues

- symptoms: "Cannot drop column from table"
  diagnosis: "Keyspaces does not support DROP COLUMN in ALTER TABLE."
  resolution: "Create a new table without the column and migrate data. Application should stop writing to the column."

- symptoms: "Unsupported data type error"
  diagnosis: "CQL data type not supported in Keyspaces (e.g., custom types, some collection nesting)."
  resolution: "Use a supported data type. Check Keyspaces documentation for supported types."

- symptoms: "Schema change stuck or not applying"
  diagnosis: "Schema changes are asynchronous. Table may be in UPDATING status."
  resolution: "Wait for table status to return to ACTIVE. Check CloudTrail for errors."

## Output Format

```yaml
root_cause: "schema_issue — <specific_cause>"
evidence:
  - type: table_schema
    content: "<current table schema>"
  - type: alter_statement
    content: "<attempted schema change>"
  - type: error_message
    content: "<specific error returned>"
severity: MEDIUM
mitigation:
  immediate: "Use supported schema operations or create new table"
  long_term: "Design schema for Keyspaces compatibility from the start"
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

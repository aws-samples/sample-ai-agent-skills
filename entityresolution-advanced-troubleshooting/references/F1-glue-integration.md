---
title: "F1 — Glue Integration"
description: "Diagnose AWS Glue integration issues including table schema, database access, and catalog configuration"
status: active
severity: HIGH
triggers: ["Glue table error", "Glue database not found", "catalog access denied", "Glue schema mismatch", "Glue integration failed"]
owner: devops-agent
objective: "Resolve Glue Data Catalog integration issues for Entity Resolution"
context: "Entity Resolution uses Glue Data Catalog for input data definition. Glue databases and tables must exist and be properly configured. The Glue table schema must match the Entity Resolution schema mapping. Glue is a core dependency."
---

## Phase 1 — Triage

MUST:
- Verify Glue database exists: `aws glue get-database --name <database-name>`
- Check Glue table exists and schema: `aws glue get-table --database-name <db> --name <table>`
- Verify service role has Glue permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names glue:GetTable glue:GetDatabase --resource-arns <glue-resource-arn>`

SHOULD:
- Compare Glue table columns with Entity Resolution schema mapping
- Check Glue table data location (S3 path) is accessible
- Verify Glue table partition configuration if data is partitioned

MAY:
- Check Glue crawler status if table is crawler-managed
- Verify Glue catalog encryption settings
- Review Glue resource policy for access restrictions

## Phase 2 — Remediate

MUST:
- Create missing Glue databases and tables
- Fix Glue table schema to match Entity Resolution schema mapping
- Grant the service role glue:GetTable, glue:GetDatabase, and glue:GetPartitions permissions

SHOULD:
- Ensure Glue table column names match schema mapping field names exactly
- Verify the S3 data location referenced by the Glue table contains data
- Update Glue table schema when input data format changes

MAY:
- Set up Glue crawlers for automatic schema detection
- Implement Glue table versioning for schema change management
- Create Glue ETL jobs for data preprocessing

## Safety Ratings

```
safety_ratings:
  - "glue get-database: GREEN — read-only database metadata"
  - "glue get-table: GREEN — read-only table schema"
  - "glue create-table: YELLOW — creates new Glue table"
  - "glue update-table: YELLOW — modifies table schema"
  - "glue delete-table: RED — permanently removes table definition"
```

## Escalation Conditions

- Glue table schema changes affect multiple Entity Resolution workflows
- Glue catalog access requires cross-account configuration
- Glue table data location has access restrictions
- Glue crawler produces unexpected schema changes

## Data Sensitivity

- `aws glue get-database`: LOW — returns database metadata
- `aws glue get-table`: LOW — returns table schema and location
- `aws iam simulate-principal-policy`: LOW — returns allow/deny results
- Glue table data: HIGH — contains input records with PII

## Prohibited Actions

- NEVER delete Glue tables referenced by active Entity Resolution workflows
- NEVER modify Glue table schema without updating corresponding schema mappings
- NEVER grant glue:* permissions — use specific actions
- NEVER point Glue tables to S3 locations containing unvalidated data

## Phase 3 — Rollback

1. If a Glue table was modified incorrectly, restore previous schema
2. If a Glue database was deleted, recreate it with the same configuration
3. Verify Entity Resolution workflows can access the restored Glue resources
4. Re-run a test matching job to validate Glue integration
5. Document the Glue configuration change and rollback

## Common Issues

- symptoms: "Workflow fails with ResourceNotFoundException for Glue table"
  diagnosis: "The Glue table or database referenced in the workflow does not exist."
  resolution: "Create the Glue database and table. Verify names match the workflow configuration exactly."

- symptoms: "Workflow runs but matching fields are empty"
  diagnosis: "Glue table column names do not match Entity Resolution schema mapping field names."
  resolution: "Compare Glue table columns with schema mapping fields. Fix naming mismatches."

- symptoms: "Glue table exists but service role cannot access it"
  diagnosis: "The service role lacks Glue permissions, or a Glue resource policy restricts access."
  resolution: "Grant glue:GetTable and glue:GetDatabase permissions. Check Glue resource policy."

## Output Format

```yaml
root_cause: "glue_integration — <specific_cause>"
evidence:
  - type: glue_error
    content: "<Glue table/database details and access issues>"
severity: HIGH
mitigation:
  immediate: "Fix Glue configuration"
  long_term: "Implement Glue schema management and monitoring"
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

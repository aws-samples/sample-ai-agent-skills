---
title: "B1 — Configured Table Issues"
description: "Diagnose Clean Rooms configured table problems including Glue catalog and association errors"
status: active
severity: HIGH
triggers:
  - "configured table"
  - "table configuration"
  - "table not found"
  - "glue table error"
  - "table association"
owner: devops-agent
objective: "Resolve configured table issues and ensure tables are available for querying"
context: "Configured tables reference Glue Data Catalog tables and must be associated with a collaboration membership. Common issues include missing Glue tables, wrong table references, association failures, and schema mismatches."
---

## Phase 1 — Triage

MUST:
- List configured tables: `aws cleanrooms list-configured-tables`
- Get table details: `aws cleanrooms get-configured-table --configured-table-identifier <id>`
- Verify Glue table exists: `aws glue get-table --database-name <db> --name <table>`
- Check table associations: `aws cleanrooms list-configured-table-associations --membership-identifier <membership-id>`

SHOULD:
- Verify Glue table schema matches expected columns
- Check S3 data location referenced by Glue table
- Verify IAM permissions for Glue and S3 access

MAY:
- Check if analysis rule is configured for the table
- Verify the table is in the correct region

## Phase 2 — Remediate

MUST:
- If Glue table missing, create it in Glue Data Catalog
- Create configured table: `aws cleanrooms create-configured-table --name <name> --table-reference '{"glue":{"tableName":"<table>","databaseName":"<db>"}}' --allowed-columns '["col1","col2"]'`
- Associate with membership: `aws cleanrooms create-configured-table-association --configured-table-identifier <table-id> --membership-identifier <membership-id> --name <name> --role-arn <service-role-arn>`

SHOULD:
- Specify allowed columns to limit data exposure
- Use a dedicated service role for Clean Rooms data access

## Output Format

```yaml
root_cause: "configured_table — <specific_cause>"
evidence:
  - type: table_config
    content: "<configured table details>"
  - type: glue_table
    content: "<Glue table details>"
severity: HIGH
mitigation:
  immediate: "Fix Glue table or configured table configuration"
  long_term: "Manage table configurations via IaC"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Create Glue table in Data Catalog | GREEN |
| Create configured table | YELLOW |
| Associate with membership | YELLOW |
| Specify allowed columns to limit data exposure | GREEN |
| Use a dedicated service role | GREEN |

## Escalation Conditions

- Fix requires modifying production table configurations
- Fix involves changing allowed columns that affect data exposure

## Data Sensitivity

- **HIGH**: configured tables reference underlying data sources and define data exposure boundaries
- Allowed columns and service role configurations control data access scope

## Prohibited Actions

- NEVER suggest disabling privacy controls
- NEVER suggest exporting raw data from collaborations

## Phase 3 — Rollback

1. If configured table was created incorrectly, delete it: `aws cleanrooms delete-configured-table --configured-table-identifier <id>`
2. If table association was misconfigured, delete and recreate the association
3. If allowed columns were changed, recreate the configured table with correct columns
4. Verify rollback by confirming `aws cleanrooms get-configured-table` shows expected configuration

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

---
title: "B1 — Schema Mapping Issues"
description: "Diagnose schema mapping configuration errors causing incorrect field mapping and poor matching results"
status: active
severity: MEDIUM
triggers: ["schema mapping error", "field mapping wrong", "columns not mapped", "schema mismatch"]
owner: devops-agent
objective: "Resolve schema mapping configuration issues"
context: "Schema mappings define how input data fields map to Entity Resolution attributes. Incorrect mappings cause poor matching results without errors — jobs complete successfully but matches are wrong or missing."
---

## Phase 1 — Triage

MUST:
- Get schema mapping details: `aws entityresolution get-schema-mapping --schema-name <schema-name>`
- Check the Glue table schema: `aws glue get-table --database-name <db> --name <table>`
- Compare schema mapping fields with Glue table columns

SHOULD:
- Verify field types match between schema mapping and Glue table
- Check if required fields are mapped (name, address, email, etc.)
- Review the matching workflow to see which schema mappings it uses

MAY:
- Sample input data to verify field content matches expectations
- Compare with a known working schema mapping

## Phase 2 — Remediate

MUST:
- Fix schema mapping field names to match Glue table columns exactly
- Ensure required matching attributes are mapped
- Verify field types are compatible

SHOULD:
- Map all relevant fields for better matching accuracy
- Use consistent naming conventions across schema mappings
- Test schema mapping changes with a sample matching job

MAY:
- Create separate schema mappings for different data sources
- Document schema mapping configurations and their purposes

## Safety Ratings

```
safety_ratings:
  - "get-schema-mapping: GREEN — read-only schema configuration"
  - "glue get-table: GREEN — read-only table schema"
  - "create-schema-mapping: YELLOW — creates new schema mapping"
  - "update-schema-mapping: YELLOW — modifies existing schema mapping"
```

## Escalation Conditions

- Schema mapping changes affect production matching workflows
- Multiple workflows share the same schema mapping
- Input data schema changes require coordinated updates
- Schema mapping complexity exceeds manageable levels

## Data Sensitivity

- `aws entityresolution get-schema-mapping`: LOW — returns mapping configuration
- `aws glue get-table`: LOW — returns table schema metadata
- Input data samples: HIGH — may contain PII
- Schema mapping configuration: LOW — contains field mapping rules

## Prohibited Actions

- NEVER modify a schema mapping used by production workflows without testing
- NEVER delete schema mappings referenced by active workflows
- NEVER map sensitive fields unnecessarily (only map fields needed for matching)
- NEVER assume field names are case-insensitive — verify exact case

## Phase 3 — Rollback

1. If a schema mapping was modified incorrectly, restore previous configuration
2. If a new schema mapping was created with errors, delete it
3. Verify workflows using the schema mapping still produce correct results
4. Re-run a test matching job to validate the rollback
5. Document the schema mapping change and rollback

## Common Issues

- symptoms: "Matching job completes but match rate is very low"
  diagnosis: "Schema mapping fields do not correctly map to Glue table columns, causing matching attributes to be empty."
  resolution: "Verify each schema mapping field name matches the exact Glue table column name. Check field content."

- symptoms: "CreateSchemaMapping fails with validation error"
  diagnosis: "The schema mapping definition has invalid field types or missing required attributes."
  resolution: "Check field type values. Ensure required attributes (like uniqueId) are defined."

- symptoms: "Schema mapping works for one table but not another"
  diagnosis: "The second Glue table has different column names or types than expected."
  resolution: "Create a separate schema mapping for each data source with different schemas."

## Output Format

```yaml
root_cause: "schema_mapping — <specific_cause>"
evidence:
  - type: mapping_error
    content: "<schema mapping vs Glue table comparison>"
severity: MEDIUM
mitigation:
  immediate: "Fix schema mapping configuration"
  long_term: "Implement schema validation in data pipeline"
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

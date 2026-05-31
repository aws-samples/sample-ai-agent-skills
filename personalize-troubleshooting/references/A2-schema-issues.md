---
title: "A2 — Schema Issues"
description: "Diagnose dataset schema configuration problems"
status: active
severity: HIGH
triggers: ["schema error", "schema mismatch", "CreateSchema error"]
owner: devops-agent
objective: "Identify and resolve schema issues"
context: "Schemas define dataset structure using Avro format. Required fields vary by dataset type. Schema must be created before dataset."
---

## Phase 1 — Triage

MUST:
- Check schema: `aws personalize describe-schema --schema-arn <arn>`
- Verify required fields for dataset type
- Check field types match data
- Verify schema is associated with correct dataset

SHOULD:
- Check for reserved field names
- Verify categorical vs numerical field types
- Check for schema version compatibility

MAY:
- Compare with working schema
- Review schema creation CloudTrail events

## Phase 2 — Remediate

MUST:
- Include required fields (USER_ID, ITEM_ID, TIMESTAMP for interactions)
- Use correct Avro types (string, long, float, etc.)
- Create schema before dataset

SHOULD:
- Mark categorical fields appropriately
- Include metadata fields for filtering

MAY:
- Implement schema validation tooling

## Common Issues

- symptoms: "CreateSchema fails with validation error"
  diagnosis: "Invalid Avro schema format."
  resolution: "Fix Avro JSON schema. Verify field types and required fields."

- symptoms: "Import fails despite correct CSV"
  diagnosis: "Schema field types don't match CSV data types."
  resolution: "Align schema types with actual data. Use 'string' for text, 'long' for timestamps."

## Output Format

```yaml
root_cause: "schema — <specific_cause>"
evidence:
  - type: schema_definition
    content: "<schema details>"
severity: HIGH
mitigation:
  immediate: "Fix schema definition"
  long_term: "Implement schema validation"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — schemas define data structure, not data content
- Service Disruption Risk: LOW — schema issues prevent import but do not affect running solutions
- Compliance Impact: LOW — schema configuration is a development concern

## Escalation Conditions
- Schema validation fails with unclear errors
- Schema changes break existing dataset imports
- Schema incompatible with required recipe
- Multiple schema versions cause confusion
- Schema does not support required data fields

## Data Sensitivity
- Schema definitions — INTERNAL
- Field names and types — INTERNAL
- Data format specifications — INTERNAL
- Schema version history — INTERNAL
- Never expose schema details that reveal data collection practices

## Prohibited Actions
- Do NOT modify schemas used by active dataset groups without migration plan
- Do NOT create schemas with PII field names that reveal data sensitivity
- Do NOT bypass schema validation
- Do NOT delete schemas referenced by active datasets
- Do NOT create overly permissive schemas
- Do NOT ignore schema compatibility warnings

## Phase 3 — Rollback
- Restore previous schema versions
- Revert schema changes that broke imports
- Re-create schemas with correct field definitions
- Verify dataset compatibility after schema changes
- Document schema issues and data format requirements
- Update schema management procedures

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

---
title: "C2 — Data Mapping Errors"
description: "Diagnose Amazon AppFlow data mapping and field transformation errors"
status: active
severity: MEDIUM
triggers: ["data mapping error", "field mapping failed", "type mismatch", "schema mismatch"]
owner: devops-agent
objective: "Resolve data mapping issues including field mismatches, type incompatibilities, and transform failures"
context: "Data mapping connects source fields to destination fields. Errors occur when schemas change, types are incompatible, or transforms fail."
---

## Phase 1 — Triage

MUST:
- Describe the flow to review mappings: `aws appflow describe-flow --flow-name <flow-name>`
- Check execution records for mapping errors: `aws appflow describe-flow-execution-records --flow-name <flow-name> --max-results 5`
- Verify source schema has not changed since flow creation

SHOULD:
- Compare source field types with destination field types
- Check if any mapped source fields were renamed or removed

MAY:
- Review the connector's supported data types: `aws appflow describe-connectors --connector-type <type>`

## Phase 2 — Remediate

MUST:
- Update field mappings to match current source schema
- Fix type incompatibilities (e.g., string to integer conversions)
- Remove mappings for deleted source fields

SHOULD:
- Add validation tasks to catch data quality issues before destination write
- Test with a small dataset after fixing mappings

MAY:
- Implement data transformation tasks (mask, truncate, arithmetic) for type conversion

## Safety Ratings

```
safety_ratings:
  - "describe-flow: GREEN — read-only flow configuration"
  - "describe-flow-execution-records: GREEN — read-only execution history"
  - "describe-connectors: GREEN — read-only connector capabilities"
  - "update-flow: YELLOW — modifies flow field mappings"
```

## Escalation Conditions

- Source schema changes frequently and breaks mappings regularly
- Complex data transformations are needed that AppFlow tasks cannot handle
- Destination schema cannot be modified to match source data types
- Mapping errors cause data corruption in the destination

## Data Sensitivity

- `aws appflow describe-flow`: MEDIUM — reveals field mapping configuration
- `aws appflow describe-flow-execution-records`: MEDIUM — reveals data processing errors
- `aws appflow describe-connectors`: LOW — reveals connector capabilities

## Prohibited Actions

- NEVER remove field validations to bypass mapping errors
- NEVER force incompatible type mappings without transformation
- NEVER modify destination schema without coordinating with downstream consumers
- NEVER assume source schema is stable — always verify before updating mappings

## Phase 3 — Rollback

1. If field mappings were changed, revert to previous mapping configuration
2. If destination schema was modified, restore original schema
3. Verify data integrity in destination after mapping changes
4. Run a test execution to confirm reverted mappings work correctly
5. Document schema change patterns for future prevention

## Common Issues

- symptoms: "ValidationException: Field 'Account.Custom_Field__c' not found in source schema"
  diagnosis: "Salesforce custom field was renamed or removed."
  resolution: "Re-fetch source schema in the flow configuration. Update field mappings to use the new field name."

## Output Format

```yaml
root_cause: "data_mapping — <specific_cause>"
evidence:
  - type: mapping_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix field mappings"
  long_term: "Monitor source schema changes"
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

---
title: "B2 — ASFF Format Issues"
description: "Diagnose AWS Security Finding Format (ASFF) errors and validation issues"
status: active
severity: MEDIUM
triggers:
  - "ASFF format"
  - "finding format error"
  - "BatchImportFindings failed"
  - "finding validation"
  - "ASFF schema"
owner: devops-agent
objective: "Identify and resolve ASFF format issues preventing finding import into Security Hub"
context: "ASFF is the mandatory format for all Security Hub findings. Import failures are typically caused by missing required fields, invalid field values, incorrect data types, or schema version mismatches. BatchImportFindings returns detailed error messages for each failed finding."
---

## Phase 1 — Triage

MUST:
- Check the BatchImportFindings error response for specific field errors
- Verify all required ASFF fields are present: SchemaVersion, Id, ProductArn, GeneratorId, AwsAccountId, Types, CreatedAt, UpdatedAt, Severity, Title, Resources
- Validate the ProductArn format: `arn:aws:securityhub:<region>:<account-id>:product/<account-id>/default`
- Check date fields are in ISO 8601 format
- Verify Severity uses the correct structure (Label or Normalized)

SHOULD:
- Validate the Resources array has at least one entry with Type, Id, and Partition
- Check Types array uses the correct taxonomy (Software and Configuration Checks, TTPs, etc.)
- Verify the SchemaVersion matches a supported version (2018-10-08)

MAY:
- Use the Security Hub API to validate finding format before batch import
- Check for field length limits (Title max 256 chars, Description max 1024 chars)

## Phase 2 — Remediate

MUST:
- Fix missing or invalid required fields
- Ensure date fields use ISO 8601 format (e.g., 2024-01-15T10:30:00.000Z)
- Validate the ProductArn matches the registered product

SHOULD:
- Create a finding template with all required fields pre-populated
- Implement validation in the finding generation pipeline
- Test with a single finding before batch import

MAY:
- Create a finding validation library for the integration
- Document the ASFF field requirements for the team

## Common Issues

- symptoms: "BatchImportFindings returns InvalidInput for ProductArn"
  diagnosis: "The ProductArn does not match a registered product or uses incorrect format."
  resolution: "Use the correct ProductArn format. For custom products, register first with CreateProduct or use the default product ARN."

- symptoms: "Finding import fails with missing required field"
  diagnosis: "One or more required ASFF fields are missing from the finding."
  resolution: "Add all required fields: SchemaVersion, Id, ProductArn, GeneratorId, AwsAccountId, Types, CreatedAt, UpdatedAt, Severity, Title, Resources."

- symptoms: "Finding import succeeds but finding not visible"
  diagnosis: "The finding may be filtered out by the current view, or the AwsAccountId does not match the importing account."
  resolution: "Check finding filters. Verify AwsAccountId matches. Check if the finding was auto-archived."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix ASFF format errors | GREEN | Data correction; no state change |
| Create finding template | GREEN | Documentation; no state change |
| Validate finding format | GREEN | Read-only validation |

## Escalation Conditions

- Cross-account configuration changes
- Custom finding import pipeline changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| BatchImportFindings error response | MEDIUM | Finding data and error details |
| Finding content | HIGH | Security finding details |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| ASFF format fix | N/A — data correction only |
| Finding import | Archive imported findings if incorrect |

## Output Format

```yaml
root_cause: "asff_format — <specific_field_error>"
evidence:
  - type: import_error
    content: "<BatchImportFindings error response>"
  - type: finding
    content: "<the finding that failed validation>"
severity: MEDIUM
mitigation:
  immediate: "Fix the ASFF format errors and retry import"
  long_term: "Implement finding validation in the generation pipeline"
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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"

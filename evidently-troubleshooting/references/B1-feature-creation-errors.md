---
title: "B1 — Feature Creation Errors"
description: "Diagnose feature creation failures in Evidently"
status: active
severity: HIGH
triggers: ["cannot create feature", "feature creation failed", "CreateFeature error", "variation error"]
owner: devops-agent
objective: "Resolve feature creation issues including variation configuration"
context: "Features define variations (boolean, string, long, double) and evaluation rules. Variation type is immutable after creation."
---

## Phase 1 — Triage

MUST:
- Check existing features in the project: `aws evidently list-features --project <name>`
- Verify IAM permissions for evidently:CreateFeature
- Confirm the feature name is unique within the project

SHOULD:
- Verify the variation type matches the application's expected data type
- Check that at least two variations are defined
- Verify the default variation is set correctly

MAY:
- Review CloudTrail for CreateFeature errors
- Check if the project exists and is in AVAILABLE status

## Phase 2 — Remediate

MUST:
- Fix variation definitions to match the correct type (boolean, string, long, double)
- Ensure at least two variations are provided
- Set the default variation to a valid variation name

SHOULD:
- Plan variation type carefully — it cannot be changed after creation
- Define meaningful variation names for clarity in launches and experiments

MAY:
- Add entity overrides during creation for testing
- Configure evaluation strategy (client-side or server-side)

## Common Issues

- symptoms: "CreateFeature returns ValidationException for variation values"
  diagnosis: "Variation values do not match the specified variation type."
  resolution: "Ensure all variation values match the type. For boolean: true/false. For string: any string. For long: integer values. For double: decimal values."

## Safety Ratings

```
safety_ratings:
  - "list-features: GREEN — read-only feature enumeration"
  - "CreateFeature: YELLOW — creates a new feature resource"
  - "get-project: GREEN — read-only project details"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- Feature creation fails with internal errors after fixing all parameters
- Feature quota limit reached for the project
- Feature creation succeeds but evaluations return unexpected results
- Multiple features need coordinated creation across projects

## Data Sensitivity

- `aws evidently list-features`: LOW — returns feature metadata
- `aws evidently CreateFeature`: LOW — creates feature with variation definitions
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER create features with placeholder variation values in production projects
- NEVER assume the variation type can be changed later — it is immutable
- NEVER create features without defining a meaningful default variation
- NEVER delete existing features to recreate with different types without stopping active launches/experiments

## Phase 3 — Rollback

1. If a feature was created with wrong configuration, delete it: `aws evidently delete-feature --project <name> --feature <feature-name>` (only if no active launches/experiments)
2. Recreate with correct variation type and values
3. If entity overrides were set incorrectly, update them: `aws evidently update-feature --project <name> --feature <feature-name> --remove-entity-overrides <entity-id>`
4. Verify the feature evaluates correctly for test entities
5. Document the creation issue and resolution

## Output Format

```yaml
root_cause: "feature_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix feature creation parameters"
  long_term: "Define feature specifications in IaC with validated variation types"
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

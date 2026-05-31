---
title: "A1 — Space Creation"
description: "Diagnose CodeCatalyst space creation failures"
status: active
severity: HIGH
triggers: ["cannot create space", "space creation failed", "space error"]
owner: devops-agent
objective: "Resolve CodeCatalyst space creation issues"
context: "Spaces are the top-level organizational unit. Creation requires a Builder ID and may require billing setup."
---

## Phase 1 — Triage

MUST:
- List existing spaces: `aws codecatalyst list-spaces`
- Verify Builder ID authentication is working
- Check if space name is already taken (names are globally unique)

SHOULD:
- Verify billing information is configured for the Builder ID
- Review CloudTrail for space creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateSpace --max-results 10`
- Check if the user has permission to create spaces

MAY:
- Verify email verification status for the Builder ID
- Check CodeCatalyst service status

## Phase 2 — Remediate

MUST:
- Use a unique space name (globally unique across all CodeCatalyst)
- Complete Builder ID email verification
- Configure billing information if required for the tier

SHOULD:
- Choose an appropriate pricing tier (Free or Enterprise)
- Set up initial space members and roles

MAY:
- Configure AWS connections during space setup
- Set up SSO for Enterprise tier spaces

## Safety Ratings

```
safety_ratings:
  - "list-spaces: GREEN — read-only enumeration"
  - "CreateSpace: YELLOW — creates organizational resource with billing implications"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "billing configuration: YELLOW — sets up payment method"
```

## Escalation Conditions

- Space creation fails with internal service errors
- Billing setup fails or is rejected
- Space name conflicts cannot be resolved
- Enterprise tier setup requires organizational approval

## Data Sensitivity

- `aws codecatalyst list-spaces`: LOW — returns space names and IDs
- Builder ID information: MEDIUM — personal identity information
- Billing information: HIGH — payment method details
- Space configuration: LOW — organizational settings

## Prohibited Actions

- NEVER create spaces without organizational approval for Enterprise tier
- NEVER share Builder ID credentials
- NEVER create duplicate spaces to work around naming conflicts
- NEVER configure billing with unauthorized payment methods

## Phase 3 — Rollback

1. If a space was created in error, verify no projects or members have been added
2. Delete the empty space if it was created incorrectly
3. Verify billing is not being charged for the deleted space
4. Document the creation attempt and any issues encountered
5. Notify the team if space creation affects organizational planning

## Common Issues

- symptoms: "Space creation fails with 'name already exists'"
  diagnosis: "Space names are globally unique across all CodeCatalyst users."
  resolution: "Choose a different, unique space name. Consider using organization prefix."

- symptoms: "Space creation requires billing but no payment method configured"
  diagnosis: "Enterprise tier or exceeding Free tier limits requires billing setup."
  resolution: "Configure billing information in CodeCatalyst settings or use Free tier."

## Output Format

```yaml
root_cause: "space_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation blocker"
  long_term: "Document space naming conventions and billing setup"
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

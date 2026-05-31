---
title: "A1 — Contact Lens Enablement"
description: "Diagnose Contact Lens enablement and activation issues"
status: active
severity: HIGH
triggers: ["Contact Lens not working", "no analytics", "no transcription", "Contact Lens not enabled"]
owner: devops-agent
objective: "Resolve Contact Lens enablement issues so analytics are generated for contacts"
context: "Contact Lens requires enablement at the instance level AND configuration in each contact flow. Both are required. The contact flow must include the 'Set recording and analytics behavior' block with Contact Lens enabled."
---

## Phase 1 — Triage

MUST:
- Check instance details: `aws connect describe-instance --instance-id <instance-id>`
- List contact flows to find the relevant flow: `aws connect list-contact-flows --instance-id <instance-id> --contact-flow-types CONTACT_FLOW --query 'ContactFlowSummaryList[*].{Id:Id,Name:Name,ContactFlowStatus:ContactFlowStatus}'`
- Verify Contact Lens is enabled at the instance level in the Connect console (Admin > Analytics > Contact Lens)

SHOULD:
- Check if the contact flow includes the "Set recording and analytics behavior" block with Contact Lens enabled
- Verify a recent contact has analytics: `aws connect describe-contact --instance-id <instance-id> --contact-id <contact-id>`

MAY:
- Review CloudTrail for instance configuration changes
- Check if the issue affects all contact flows or specific ones

## Phase 2 — Remediate

MUST:
- Enable Contact Lens at the instance level via the Connect console
- Add "Set recording and analytics behavior" block to the contact flow with Contact Lens analytics enabled
- Publish the updated contact flow

SHOULD:
- Configure the language in the analytics block to match the expected call language
- Enable both real-time and post-contact analytics
- Test with a new contact after enabling

MAY:
- Enable Contact Lens for all contact flows using a common module
- Set up CloudWatch alarms for Contact Lens processing errors

## Safety Ratings

```
safety_ratings:
  - "describe-instance: GREEN — read-only instance details"
  - "list-contact-flows: GREEN — read-only flow enumeration"
  - "describe-contact: GREEN — read-only contact details"
  - "update-contact-flow-content: RED — modifies live contact flow routing"
  - "enable Contact Lens: YELLOW — enables analytics processing, incurs costs"
```

## Escalation Conditions

- Contact Lens enablement option not available in the Connect console
- Contact Lens enabled but no analytics generated after multiple test contacts
- Instance-level enablement requires organization-level approval
- Contact Lens processing errors in CloudWatch logs

## Data Sensitivity

- `aws connect describe-instance`: LOW — returns instance metadata
- `aws connect list-contact-flows`: LOW — returns flow names and IDs
- `aws connect describe-contact`: MEDIUM — returns contact metadata including agent and queue info
- Contact flow content: MEDIUM — reveals routing logic and business rules

## Prohibited Actions

- NEVER modify production contact flows without testing in a non-production environment first
- NEVER enable Contact Lens without understanding the cost implications
- NEVER disable call recording when enabling Contact Lens analytics
- NEVER assume Contact Lens is enabled just because the instance exists

## Phase 3 — Rollback

1. If Contact Lens was enabled in error, disable it at the instance level via the Connect console
2. Revert contact flow changes by publishing the previous version
3. Verify call routing is not affected by contact flow changes
4. Check that existing call recording is still functioning
5. Document the enablement attempt and reason for rollback

## Common Issues

- symptoms: "Contact Lens enabled at instance level but no analytics for contacts"
  diagnosis: "Contact flow does not include the 'Set recording and analytics behavior' block with Contact Lens enabled."
  resolution: "Add the analytics block to the contact flow and publish it."

## Output Format

```yaml
root_cause: "contact_lens_enablement — <specific_cause>"
evidence:
  - type: enablement_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Enable Contact Lens at both instance and contact flow levels"
  long_term: "Use contact flow modules for consistent analytics configuration"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"

---
title: "E2 — Journaling Rules"
description: "Diagnose WorkMail journaling rule issues"
status: active
severity: MEDIUM
triggers: ["journaling not working", "journal email missing", "compliance recording failed", "journal rule error"]
owner: devops-agent
objective: "Resolve journaling rule issues including configuration, delivery, and compliance requirements"
context: "Journaling captures copies of all sent and received email for compliance. Journal email is delivered to a specified email address."
---

## Phase 1 — Triage

MUST:
- Check organization journaling configuration: `aws workmail describe-organization --organization-id <org-id>`
- Verify the journal recipient email address is valid and accessible
- Check if journaling is enabled for the organization
- Verify the journal recipient mailbox is not full

SHOULD:
- Check if email flow rules are interfering with journal delivery
- Verify the journal recipient is receiving email from other sources
- Check CloudTrail for journaling configuration changes

MAY:
- Review journal email volume to estimate storage requirements
- Check if specific email types are excluded from journaling

## Phase 2 — Remediate

MUST:
- Configure journaling with a valid recipient email address
- Ensure the journal recipient mailbox has sufficient capacity
- Fix email flow rules that may block journal delivery

SHOULD:
- Use a dedicated mailbox or external archival system for journal email
- Monitor journal delivery to ensure compliance requirements are met

MAY:
- Set up alerts for journal delivery failures
- Configure retention policies on the journal mailbox

## Safety Ratings

```
safety_ratings:
  - "workmail describe-organization: GREEN — read-only organization inspection"
  - "Journaling configuration: YELLOW — affects compliance email capture"
  - "Journal recipient changes: RED — may cause compliance gaps if misconfigured"
  - "Disable journaling: RED — stops compliance email capture"
```

## Escalation Conditions

- Journaling is required for compliance but not functioning
- Journal email delivery has gaps that may violate regulatory requirements
- Journal mailbox is full and new journal email is being bounced
- Journaling configuration cannot be changed due to admin restrictions

## Data Sensitivity

- `aws workmail describe-organization`: LOW — returns organization metadata
- Journal email content: HIGH — contains copies of all organizational email
- Journal recipient address: MEDIUM — reveals compliance infrastructure
- Journaling configuration: MEDIUM — reveals compliance posture

## Prohibited Actions

- NEVER disable journaling without compliance team approval
- NEVER delete journal email without verifying retention requirements
- NEVER redirect journal email to a personal mailbox
- NEVER expose journal email content in troubleshooting logs

## Phase 3 — Rollback

1. If journaling was disabled, re-enable it immediately and document the gap period
2. If the journal recipient was changed, restore the original recipient address
3. If email flow rules affecting journaling were modified, restore original rules
4. Notify the compliance team of any journaling interruptions
5. Verify journal email is being delivered correctly after changes

## Common Issues

- symptoms: "Journal email not being delivered"
  diagnosis: "Journal recipient email address is invalid or mailbox is full."
  resolution: "Verify journal recipient address. Ensure recipient mailbox has capacity. Use external archival if needed."

- symptoms: "Journaling enabled but some emails not captured"
  diagnosis: "Email flow rules may be routing some email before journaling captures it."
  resolution: "Review email flow rules. Ensure journaling captures email at the correct point in the flow."

## Output Format

```yaml
root_cause: "journaling — <specific_cause>"
evidence:
  - type: journaling_issue
    content: "<journaling configuration and delivery status>"
severity: MEDIUM
mitigation:
  immediate: "Fix journaling configuration"
  long_term: "Monitor journal delivery and set up compliance alerts"
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

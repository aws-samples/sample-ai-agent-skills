---
title: "B1 — Contact Channel Issues"
description: "Diagnose Incident Manager contact channel problems including activation and delivery"
status: active
severity: HIGH
triggers:
  - "contact channel"
  - "notification not received"
  - "sms not delivered"
  - "email not received"
  - "channel activation"
owner: devops-agent
objective: "Resolve contact channel delivery issues and ensure notifications reach responders"
context: "Contact channels (email, SMS, voice) must be activated by recipients before they can receive notifications. Common issues include unactivated channels, incorrect contact info, SNS delivery failures, and channel type limitations."
---

## Phase 1 — Triage

MUST:
- Get contact details: `aws ssm-contacts get-contact --contact-id <contact-arn>`
- List contact channels: `aws ssm-contacts list-contact-channels --contact-id <contact-arn>`
- Check channel activation status for each channel
- Verify contact information (email, phone number) is correct

SHOULD:
- Check SNS delivery logs if using SNS-backed channels
- Verify the contact is part of an active engagement
- Check if the channel type is supported in the region

MAY:
- Send a test engagement: `aws ssm-contacts start-engagement --contact-id <contact-arn> --subject "Test" --content "Test notification"`
- Check CloudTrail for engagement events

## Phase 2 — Remediate

MUST:
- If channel not activated, send activation: `aws ssm-contacts activate-contact-channel --contact-channel-id <channel-arn> --activation-code <code>`
- If contact info wrong, update channel with correct details
- If channel missing, create it: `aws ssm-contacts create-contact-channel --contact-id <contact-arn> --name <name> --type <EMAIL|SMS|VOICE> --delivery-address '{"SimpleAddress":"<address>"}'`

SHOULD:
- Configure multiple channel types per contact for redundancy
- Test channels after activation

## Output Format

```yaml
root_cause: "contact_channel — <specific_cause>"
evidence:
  - type: channel_status
    content: "<contact channel details>"
  - type: activation_status
    content: "<activation status>"
severity: HIGH
mitigation:
  immediate: "Activate channels or fix contact information"
  long_term: "Regularly verify channel activation and test delivery"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete response plans that are actively referenced by CloudWatch alarms
- Never modify escalation plans during active incidents
- Never remove contact channels without confirming alternative notification paths exist
- Never delete incident records or post-incident analysis data
- Never disable automation runbooks tied to active response plans without backup procedures

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous response plan configuration if incident detection breaks
- Revert escalation plan changes if notifications fail to reach responders
- Re-enable previous contact channel settings if engagement plans stop working

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

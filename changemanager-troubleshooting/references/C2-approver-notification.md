---
title: "C2 — Approver Notification"
description: "Diagnose approver notification delivery issues"
status: active
severity: MEDIUM
triggers: ["approver not notified", "no notification", "SNS not working"]
owner: devops-agent
objective: "Resolve approver notification delivery"
context: "Notifications are sent via SNS. Approvers must be subscribed to the SNS topic configured in Change Manager."
---

## Phase 1 — Triage

MUST:
- Check SNS topic configuration in Change Manager settings
- Verify SNS topic exists: `aws sns get-topic-attributes --topic-arn <arn>`
- Check SNS subscriptions: `aws sns list-subscriptions-by-topic --topic-arn <arn>`

SHOULD:
- Verify approver email/phone is subscribed and confirmed
- Check SNS delivery logs for failures

MAY:
- Test SNS delivery: `aws sns publish --topic-arn <arn> --message "test"`

## Phase 2 — Remediate

MUST:
- Configure valid SNS topic in Change Manager
- Subscribe approvers to the SNS topic
- Ensure subscriptions are confirmed

SHOULD:
- Use email subscriptions for reliability
- Set up delivery status logging

MAY:
- Add multiple notification channels (email + SMS)

## Common Issues

- symptoms: "Approvers say they never received notification"
  diagnosis: "SNS subscription not confirmed or email in spam."
  resolution: "Verify subscription is confirmed. Check spam folders. Resend confirmation."

## Output Format

```yaml
root_cause: "approver_notification — <specific_cause>"
evidence:
  - type: sns_config
    content: "<SNS topic and subscription status>"
severity: MEDIUM
mitigation:
  immediate: "Fix notification delivery"
  long_term: "Verify notification delivery as part of template setup"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Configure valid SNS topic in Change Manager | GREEN |
| Subscribe approvers to the SNS topic | GREEN |
| Ensure subscriptions are confirmed | GREEN |
| Use email subscriptions for reliability | GREEN |
| Set up delivery status logging | GREEN |
| Add multiple notification channels (email + SMS) | GREEN |

## Escalation Conditions

- Fix requires bypassing approval workflow
- Change affects production infrastructure

## Data Sensitivity

- **MEDIUM**: change request details, approval workflows
- SNS topic subscriptions reveal approver contact information

## Prohibited Actions

- NEVER suggest bypassing approval workflows

## Phase 3 — Rollback

1. If SNS topic was changed, restore the previous topic ARN in Change Manager settings
2. If subscriptions were added, remove incorrect subscriptions from the SNS topic
3. If delivery status logging was enabled, disable it if not needed
4. Verify rollback by confirming `aws sns list-subscriptions-by-topic` shows expected subscribers

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

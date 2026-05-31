---
title: "A2 — Action Approval Issues"
description: "Diagnose AWS Chatbot action approval workflow failures including approver permissions and timeout issues"
status: active
severity: HIGH
triggers: ["approval not working", "cannot approve action", "approval timeout", "action pending approval"]
owner: devops-agent
objective: "Resolve action approval workflow issues in AWS Chatbot"
context: "Action approvals require specific IAM permissions for approvers. Approvals are channel-specific and governed by guardrail configuration. Approval timeouts cause action rejection."
---

## Phase 1 — Triage

MUST:
- Check channel configuration for approval settings: `aws chatbot describe-slack-channel-configurations --chat-configuration-arn <arn>`
- Verify approver IAM permissions
- Check if the action requires approval based on guardrail configuration

SHOULD:
- Review CloudTrail for approval-related events
- Check if the approval timeout is configured appropriately
- Verify the approver is a member of the chat channel

MAY:
- Test approval flow with a non-sensitive action
- Review notification delivery for approval requests

## Phase 2 — Remediate

MUST:
- Grant approvers the required IAM permissions for Chatbot approval actions
- Configure appropriate approval timeouts
- Ensure approval notifications reach the designated approvers

SHOULD:
- Define clear approval policies in guardrails
- Set up multiple approvers to avoid single-point-of-failure
- Document which actions require approval and who can approve

MAY:
- Implement escalation workflows for unanswered approvals
- Configure approval notifications to additional channels

## Safety Ratings

```
safety_ratings:
  - "describe-slack-channel-configurations: GREEN — read-only configuration"
  - "cloudtrail lookup-events: GREEN — read-only audit logs"
  - "update-slack-channel-configuration: YELLOW — modifies approval settings"
  - "Approving actions: YELLOW to RED — depends on the action being approved"
```

## Escalation Conditions

- Critical action blocked by approval with no available approvers
- Approval permissions require organizational IAM policy changes
- Approval timeout too short for organizational processes
- Approval workflow blocks incident response actions

## Data Sensitivity

- `aws chatbot describe-slack-channel-configurations`: LOW — configuration metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals who approved/rejected actions
- Approval context: MEDIUM — may contain details about the action being approved
- Channel membership: LOW — reveals who has access to the channel

## Prohibited Actions

- NEVER bypass approval workflows for high-risk actions
- NEVER grant approval permissions to all channel members without review
- NEVER set approval timeout to zero to skip approvals
- NEVER approve actions without understanding what they will do

## Phase 3 — Rollback

1. If an action was incorrectly approved, attempt to stop the underlying operation
2. Revert any changes made by the approved action
3. If approval configuration was changed, restore previous settings
4. Review and update approval policies based on the incident
5. Notify affected teams about the approval issue

## Common Issues

- symptoms: "Approver clicks approve but gets permission denied"
  diagnosis: "The approver's IAM principal lacks Chatbot approval permissions."
  resolution: "Grant the approver chatbot:* or specific Chatbot approval permissions."

- symptoms: "Action times out waiting for approval"
  diagnosis: "No approver responded within the configured timeout period."
  resolution: "Increase timeout. Add more approvers. Set up approval notifications."

- symptoms: "Action does not prompt for approval when it should"
  diagnosis: "The guardrail configuration does not require approval for this action type."
  resolution: "Update guardrail policy to require approval for the specific action or command."

## Output Format

```yaml
root_cause: "action_approval — <specific_cause>"
evidence:
  - type: approval_error
    content: "<approval failure details>"
severity: HIGH
mitigation:
  immediate: "Fix approval configuration"
  long_term: "Establish clear approval policies and escalation"
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

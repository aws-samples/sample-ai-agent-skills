---
title: "H2 — Lifecycle Event Issues"
description: "Diagnose issues with Control Tower lifecycle events and automation"
status: active
severity: MEDIUM
triggers:
  - "Lifecycle event not firing"
  - "CreateManagedAccount event"
  - "EventBridge rule not triggering"
  - "Post-account automation failed"
owner: devops-agent
objective: "Resolve lifecycle event delivery and processing issues"
context: "Control Tower emits lifecycle events via EventBridge when accounts are created, enrolled, or updated. These events trigger automation (Lambda, Step Functions, etc.) for post-provisioning tasks. Issues arise from EventBridge rule misconfigurations, Lambda failures, or event format changes."
---

## Phase 1 — Triage

MUST:
- Check EventBridge rules for CT lifecycle events: `aws events list-rules --name-prefix "controltower"`
- Verify the rule pattern matches CT event format
- Check the rule target (Lambda, SNS, Step Functions) status
- Check CloudWatch Logs for the target function/workflow

SHOULD:
- Verify the event was emitted by checking CloudTrail
- Check if the EventBridge rule is enabled
- Verify the target has correct permissions (resource-based policy)

MAY:
- Test the EventBridge rule with a test event
- Check if the event format changed after a landing zone update

## Phase 2 — Remediate

MUST:
- Fix the EventBridge rule pattern if it doesn't match CT events
- Fix the target function/workflow if it's failing
- Verify the target has permission to be invoked by EventBridge

SHOULD:
- Add error handling and DLQ to the target function
- Implement idempotent processing (events may be delivered more than once)
- Test the full lifecycle event flow end-to-end

MAY:
- Set up monitoring for lifecycle event processing failures
- Implement retry logic for transient failures

## Common Issues

- symptoms: "Automation doesn't run when new account is created"
  diagnosis: "EventBridge rule pattern doesn't match the CT lifecycle event."
  resolution: "Update the rule pattern to match the CreateManagedAccount event format."

- symptoms: "Lambda target fails with permission error"
  diagnosis: "Lambda resource-based policy doesn't allow EventBridge invocation."
  resolution: "Add permission for events.amazonaws.com to invoke the Lambda function."

- symptoms: "Lifecycle events stopped after landing zone update"
  diagnosis: "Event format or source changed in the new CT version."
  resolution: "Review the updated event format and adjust EventBridge rules."

## Safety Ratings

```
safety_ratings:
  - "list-rules: GREEN — read-only EventBridge rule listing"
  - "describe-rule: GREEN — read-only rule configuration"
  - "update EventBridge rule pattern: YELLOW — recoverable, rule can be reverted"
  - "add Lambda permission for EventBridge: YELLOW — recoverable, permission can be removed"
  - "modify lifecycle automation logic: YELLOW — recoverable, code changes are version-controlled"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Lifecycle event format changed after landing zone update
- Automation failure caused accounts to be provisioned without post-provisioning configuration
- Drift affects multiple OUs or accounts

## Data Sensitivity

- HIGH: Organization account list (all account IDs and emails)
- HIGH: SCP policy documents (expose security boundaries)
- MEDIUM: OU structure, control enablement status
- MEDIUM: Lifecycle event payloads (contain account IDs, OU details)
- MEDIUM: Lambda function code and configuration (expose automation logic)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If EventBridge rule changes break automation, revert the rule pattern to the previous version
- If Lambda function changes cause failures, deploy the previous function version
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- Document the EventBridge rule pattern and target configuration before making changes
- If lifecycle automation was broken, manually run the post-provisioning steps for any accounts created during the outage

## Output Format

```yaml
root_cause: "lifecycle_event — <specific issue>"
evidence:
  - type: eventbridge_rule
    content: "<rule configuration and status>"
severity: MEDIUM
mitigation:
  immediate: "Fix the EventBridge rule or target"
  long_term: "Implement monitoring and testing for lifecycle event automation"
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
  - "NEVER suggest disabling mandatory controls"
  - "NEVER suggest removing the audit account"
  - "NEVER modify landing zone without understanding blast radius"

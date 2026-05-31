---
title: "C1 — Approval Workflow Failures"
description: "Diagnose SSM Automation approval step failures including timeouts, notification issues, and approver configuration"
status: active
severity: HIGH
triggers: ["approval timeout", "approval step failed", "no approvers", "approval not received", "waiting for approval"]
owner: devops-agent
objective: "Resolve approval workflow failures and ensure approvals are processed correctly"
context: "The aws:approve step type pauses execution until manual approval is received. Approvals have configurable timeouts, approver lists, and SNS notification integration."
---

## Phase 1 — Triage

MUST:
- Get execution details to find the approval step: `aws ssm get-automation-execution --automation-execution-id <execution-id>`
- Check step status: `aws ssm describe-automation-step-executions --automation-execution-id <execution-id> --filters Key=StepName,Values=<approval-step-name>`
- Verify the approval step configuration in the document: `aws ssm get-document --name <document-name>`

SHOULD:
- Check if SNS notifications were sent: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`
- Verify approver IAM permissions for ssm:SendAutomationSignal
- Review the timeout configuration for the approval step

MAY:
- Check CloudTrail for SendAutomationSignal events
- Review SNS delivery logs for notification failures

## Phase 2 — Remediate

MUST:
- Send approval signal if execution is still waiting: `aws ssm send-automation-signal --automation-execution-id <execution-id> --signal-type Approve`
- Fix approver IAM permissions if they cannot approve
- Correct SNS topic configuration for approval notifications

SHOULD:
- Increase timeout if approvals consistently take longer than configured
- Add multiple approvers to avoid single-point-of-failure
- Configure SNS notifications to alert approvers promptly

MAY:
- Implement approval automation via Lambda for low-risk changes
- Set up escalation workflows for unanswered approvals

## Safety Ratings

```
safety_ratings:
  - "describe-automation-step-executions: GREEN — read-only step details"
  - "send-automation-signal Approve: YELLOW — approves execution to proceed with changes"
  - "send-automation-signal Reject: YELLOW — rejects and stops execution"
  - "stop-automation-execution: RED — forcefully cancels execution"
```

## Escalation Conditions

- Approval step blocks a critical production change with no available approvers
- SNS notifications consistently fail to reach approvers
- Approval timeout is too short for organizational approval processes
- Multiple approval steps in sequence create excessive delays

## Data Sensitivity

- `aws ssm get-automation-execution`: MEDIUM — may contain execution parameters
- `aws ssm send-automation-signal`: MEDIUM — triggers execution to proceed
- `aws sns list-subscriptions-by-topic`: LOW — returns subscription metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals who approved/rejected and when

## Prohibited Actions

- NEVER approve an automation execution without understanding what changes it will make
- NEVER bypass approval steps by modifying the document to remove them
- NEVER send reject signals to executions without notifying the execution owner
- NEVER configure auto-approval mechanisms for high-risk production changes

## Phase 3 — Rollback

1. If an execution was incorrectly approved, stop it immediately: `aws ssm stop-automation-execution --automation-execution-id <id> --type Cancel`
2. Review any changes made by steps that executed after the approval
3. Revert resource changes made by post-approval steps
4. If SNS topic was misconfigured, restore previous configuration
5. Document the incident and update approval procedures

## Common Issues

- symptoms: "Approval step times out after 7 days with no response"
  diagnosis: "Approvers were not notified or the timeout is too short for the approval process."
  resolution: "Configure SNS notifications. Increase timeoutSeconds. Add multiple approvers."

- symptoms: "Approver gets AccessDeniedException when trying to approve"
  diagnosis: "The approver's IAM principal lacks ssm:SendAutomationSignal permission."
  resolution: "Grant the approver ssm:SendAutomationSignal permission on the automation execution resource."

- symptoms: "Approval notification not received"
  diagnosis: "SNS topic is misconfigured or the subscription is not confirmed."
  resolution: "Verify SNS topic ARN in the document. Check subscription status. Confirm pending subscriptions."

## Output Format

```yaml
root_cause: "approval_workflow — <specific_cause>"
evidence:
  - type: approval_error
    content: "<approval step status and failure details>"
severity: HIGH
mitigation:
  immediate: "Resolve approval blockage"
  long_term: "Improve approval notification and escalation"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"

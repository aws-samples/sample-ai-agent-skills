---
title: "A3 — Approval Timeout"
description: "Diagnose and resolve CodePipeline manual approval stage timeouts"
status: active
severity: MEDIUM
triggers:
  - "approval timeout"
  - "approval expired"
  - "waiting for approval"
  - "manual approval"
owner: devops-agent
objective: "Identify why approval actions are timing out and resolve notification or configuration issues"
context: "Manual approval actions pause the pipeline until someone approves or rejects. The default timeout is 7 days. Timeouts often occur because approval notifications are not reaching the right people, or the approval URL is not accessible."
---

## Phase 1 — Triage

MUST:
- Check pipeline state for approval action status: `aws codepipeline get-pipeline-state --name <pipeline-name>`
- Get approval action configuration: `aws codepipeline get-pipeline --name <pipeline-name> --query 'pipeline.stages[].actions[?actionTypeId.category==`Approval`]'`
- Check if SNS topic is configured for approval notifications: look for `NotificationArn` in the action configuration
- If SNS configured, verify topic exists and has subscriptions: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`

SHOULD:
- Check if approval notification emails are being delivered (check SNS delivery logs)
- Verify the approval URL is accessible (if custom URL configured)
- Check IAM permissions for who can approve: `codepipeline:PutApprovalResult` permission required

MAY:
- Check CloudTrail for PutApprovalResult events to see if approvals were attempted
- Review approval action comments and custom data configuration
- Check if the pipeline execution was superseded while waiting for approval

## Phase 2 — Remediate

MUST:
- If no SNS topic configured: add an SNS topic to the approval action for notifications
- If SNS topic has no subscriptions: add email or other subscriptions to the topic
- If approval timed out: start a new pipeline execution after fixing notification issues
- If approver lacks permissions: grant `codepipeline:PutApprovalResult` to the approver's IAM role

SHOULD:
- Configure approval notifications to go to a team distribution list, not individuals
- Set appropriate timeout values based on team response time expectations
- Add custom review URL pointing to the change details (PR, diff, test results)

MAY:
- Automate approvals for non-production environments using Lambda
- Integrate with Slack or other chat tools for approval notifications via SNS → Lambda → Slack

## Common Issues

- symptoms: "Approval action timed out after 7 days"
  diagnosis: "No one approved or rejected within the timeout period. Likely notification issue."
  resolution: "Verify SNS topic has active subscriptions. Check spam folders. Consider shorter timeout with re-trigger."

- symptoms: "Approval notification not received"
  diagnosis: "SNS topic not configured, subscription not confirmed, or email in spam."
  resolution: "Add SNS topic to approval action. Confirm all subscriptions. Check SNS delivery status logging."

- symptoms: "User cannot approve — Access Denied"
  diagnosis: "Approver's IAM role lacks codepipeline:PutApprovalResult permission."
  resolution: "Add codepipeline:PutApprovalResult permission to the approver's IAM policy."

- symptoms: "Pipeline stuck at approval but no one knows about it"
  diagnosis: "No notification mechanism configured for the approval action."
  resolution: "Configure NotificationArn with an SNS topic. Add CloudWatch Event rule for pipeline state changes."

## Output Format

```yaml
root_cause: "Approval timeout — <specific_cause>"
evidence:
  - type: pipeline_state
    content: "<approval action status and timeout details>"
  - type: sns_config
    content: "<SNS topic and subscription status>"
severity: MEDIUM
mitigation:
  immediate: "Fix notification configuration and start new pipeline execution"
  long_term: "Configure SNS notifications, set appropriate timeouts, consider automated approvals for lower environments"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| If no SNS topic configured: add an SNS topic to the approval action | GREEN |
| If SNS topic has no subscriptions: add subscriptions | GREEN |
| If approval timed out: start a new pipeline execution | GREEN |
| If approver lacks permissions: grant codepipeline:PutApprovalResult | YELLOW |
| Configure approval notifications to go to a team distribution list | GREEN |
| Set appropriate timeout values | GREEN |
| Add custom review URL | GREEN |
| Automate approvals for non-production environments using Lambda | YELLOW |
| Integrate with Slack or other chat tools for approval notifications | GREEN |

## Escalation Conditions

- Fix requires modifying production pipeline
- Fix requires changing deployment strategy

## Data Sensitivity

- **HIGH**: build logs (may contain secrets), artifact contents
- Approval configurations may reveal deployment gates and authorization requirements

## Prohibited Actions

- NEVER suggest storing secrets in buildspec.yml
- NEVER suggest skipping approval stages in production pipelines

## Phase 3 — Rollback

1. If SNS topic was added to approval action, remove it if it was incorrect
2. If approver IAM permissions were modified, revert to the previous policy
3. If automated approval Lambda was created, disable or delete it
4. If timeout values were changed, restore the previous timeout configuration
5. Verify rollback by confirming pipeline approval action configuration matches expected state

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

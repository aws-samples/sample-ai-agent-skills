---
title: "E1 — Notification Failures"
description: "Diagnose Elastic Transcoder SNS notification delivery failures"
status: active
severity: MEDIUM
triggers: ["no notifications", "SNS not working", "notification not received", "pipeline notifications"]
owner: devops-agent
objective: "Resolve SNS notification delivery issues for pipeline job status events"
context: "Pipelines can send SNS notifications for Progressing, Completed, Warning, and Error job states. Both IAM role and SNS topic policy must allow publishing."
---

## Phase 1 — Triage

MUST:
- Read pipeline notification configuration: `aws elastictranscoder read-pipeline --id <pipeline-id>` — check Notifications section
- Verify SNS topics exist: `aws sns get-topic-attributes --topic-arn <topic-arn>`
- Verify IAM role has sns:Publish permission: `aws iam get-role-policy --role-name <role-name> --policy-name <policy-name>`
- Check SNS topic subscriptions: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`

SHOULD:
- Verify SNS topic policy allows elastictranscoder.amazonaws.com to publish
- Check if subscriptions are confirmed (PendingConfirmation vs Confirmed)
- Test SNS topic independently: `aws sns publish --topic-arn <topic-arn> --message "test"`

MAY:
- Check CloudWatch metrics for SNS delivery failures
- Review CloudTrail for Publish events to the SNS topic

## Phase 2 — Remediate

MUST:
- Configure SNS topic ARNs in pipeline notification settings
- Add sns:Publish permission to the pipeline IAM role
- Update SNS topic policy to allow the Elastic Transcoder service to publish

SHOULD:
- Configure notifications for all job states (Progressing, Completed, Warning, Error)
- Confirm all SNS subscriptions

MAY:
- Set up CloudWatch alarms for SNS delivery failures
- Use SNS message filtering for selective notification routing

## Safety Ratings

```
safety_ratings:
  - "read-pipeline: GREEN — read-only pipeline inspection"
  - "sns get-topic-attributes: GREEN — read-only topic inspection"
  - "sns publish (test): YELLOW — sends a test message to subscribers"
  - "update-pipeline (notifications): YELLOW — modifies pipeline notification config"
```

## Escalation Conditions

- SNS notifications fail despite correct IAM and topic policy configuration
- Notifications are delivered for some job states but not others
- SNS topic is in a different account and cross-account publishing fails
- Notification delivery latency exceeds acceptable thresholds

## Data Sensitivity

- `aws elastictranscoder read-pipeline`: LOW — returns pipeline configuration
- `aws sns get-topic-attributes`: LOW — returns topic metadata and policy
- `aws sns list-subscriptions-by-topic`: MEDIUM — reveals subscriber endpoints (email, HTTP, Lambda)
- `aws sns publish`: LOW — sends test message only

## Prohibited Actions

- NEVER delete SNS topics that may be used by other services
- NEVER remove notification configuration from a production pipeline without approval
- NEVER publish test messages to production SNS topics without warning subscribers
- NEVER modify SNS topic policies without understanding the impact on other publishers

## Phase 3 — Rollback

1. If pipeline notification configuration was changed, restore original topic ARNs: `aws elastictranscoder update-pipeline-notifications --id <pipeline-id> --notifications <original-config>`
2. If SNS topic policy was modified, restore the original policy: `aws sns set-topic-attributes --topic-arn <arn> --attribute-name Policy --attribute-value <original-policy>`
3. If IAM role policy was changed, revert the policy to its original state
4. Verify notifications are flowing correctly after rollback
5. Document the notification configuration for future reference

## Common Issues

- symptoms: "No notifications received for completed jobs"
  diagnosis: "Pipeline notification configuration has empty or missing topic ARNs for Completed state."
  resolution: "Update pipeline notifications to include SNS topic ARN for Completed state."

- symptoms: "SNS publish returns AuthorizationError"
  diagnosis: "IAM role lacks sns:Publish permission or SNS topic policy blocks the service."
  resolution: "Add sns:Publish to IAM role and update SNS topic policy to allow elastictranscoder.amazonaws.com."

## Output Format

```yaml
root_cause: "notification_failure — <specific_cause>"
evidence:
  - type: notification_config
    content: "<pipeline notification settings and SNS topic details>"
severity: MEDIUM
mitigation:
  immediate: "Fix notification configuration"
  long_term: "Monitor SNS delivery with CloudWatch"
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

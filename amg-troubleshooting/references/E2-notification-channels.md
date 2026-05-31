---
title: "E2 — Notification Channel Issues"
description: "Diagnose alert notification delivery failures in Amazon Managed Grafana"
status: active
severity: HIGH
triggers:
  - "notification not received"
  - "SNS notification"
  - "contact point"
  - "alert notification failed"
  - "notification channel"
owner: devops-agent
objective: "Identify why alert notifications fail to deliver and restore notification flow"
context: "AMG supports notification channels (contact points) including Amazon SNS, Slack, PagerDuty, and others. SNS is the primary AWS-native notification destination and must be enabled on the workspace. Notification delivery failures can stem from workspace configuration, contact point misconfiguration, IAM permissions, or notification policy routing."
---

## Phase 1 — Triage

MUST:
- Verify notification destinations are enabled on the workspace: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.notificationDestinations'`
- List contact points via Grafana API: `curl -H "Authorization: Bearer <api-key>" https://<workspace-endpoint>/api/v1/provisioning/contact-points`
- Check notification policies for routing rules: `curl -H "Authorization: Bearer <api-key>" https://<workspace-endpoint>/api/v1/provisioning/policies`
- Verify the workspace IAM role has SNS publish permissions if using SNS: `aws iam list-attached-role-policies --role-name <workspace-role-name>`

SHOULD:
- Test the contact point by sending a test notification from the Grafana UI
- Check SNS topic subscription status: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`
- Verify the SNS topic policy allows the workspace role to publish

MAY:
- Check CloudTrail for SNS publish events
- Review notification templates for formatting issues

## Phase 2 — Remediate

MUST:
- Enable SNS as a notification destination on the workspace if not enabled
- Add sns:Publish permission to the workspace IAM role for the target SNS topic
- Configure the contact point with the correct SNS topic ARN

SHOULD:
- Set up a default notification policy to catch alerts without specific routing
- Verify SNS topic subscriptions are confirmed (not pending)
- Test end-to-end notification flow with a test alert

MAY:
- Configure multiple contact points for different alert severities
- Set up notification templates for consistent alert message formatting

## Common Issues

- symptoms: "Alert fires but no SNS notification is sent"
  diagnosis: "SNS is not enabled as a notification destination on the workspace, or the workspace IAM role lacks sns:Publish permission."
  resolution: "Enable SNS on the workspace and add sns:Publish permission to the workspace IAM role."

- symptoms: "SNS notification sent but not received by subscribers"
  diagnosis: "SNS topic subscription is in 'Pending' state or the subscriber endpoint is incorrect."
  resolution: "Confirm the SNS subscription. Verify the subscriber endpoint (email, Lambda, SQS) is correct."

- symptoms: "Notification policy does not route alerts to the correct contact point"
  diagnosis: "Notification policy labels do not match the alert rule labels."
  resolution: "Update notification policy matchers to match the alert rule labels, or use the default policy."

## Safety Ratings

```
safety_ratings:
  - "describe-workspace (notificationDestinations): GREEN — read-only check"
  - "list contact points via API: GREEN — read-only enumeration"
  - "add sns:Publish to workspace role: YELLOW — grants SNS publish access"
  - "enable SNS on workspace: YELLOW — changes workspace configuration"
  - "send test notification: YELLOW — sends actual notification to subscribers"
```

## Escalation Conditions

- Production alerts firing but no notifications reaching on-call teams
- SNS topic permissions require cross-account or cross-team coordination
- Notification failures affect multiple contact points simultaneously
- SNS subscription confirmation pending for critical notification endpoints

## Data Sensitivity

- Contact point configuration: MEDIUM — reveals notification endpoints (emails, webhooks)
- SNS topic subscriptions: MEDIUM — reveals who receives alert notifications
- Notification policy routing: MEDIUM — reveals alert routing logic

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER remove the default notification policy without a replacement
- NEVER send test notifications to production SNS topics during business hours without warning

## Phase 3 — Rollback

1. If SNS was enabled on the workspace incorrectly, it can remain enabled (no harm)
2. If IAM permissions were changed, detach incorrect policies from the workspace role
3. If contact points were modified, restore previous configuration via Grafana API
4. If notification policies were changed, restore previous routing rules
5. Verify end-to-end notification delivery after rollback

## Output Format

```yaml
root_cause: "notification_channel — <specific_cause>"
evidence:
  - type: workspace_config
    content: "<notification destinations enabled>"
  - type: contact_point
    content: "<contact point configuration>"
  - type: notification_policy
    content: "<routing rules and label matchers>"
severity: HIGH
mitigation:
  immediate: "Fix notification destination, contact point, or routing configuration"
  long_term: "Implement notification testing and monitoring for delivery failures"
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

---
title: "A1 — SNS Delivery Failures"
description: "Diagnose message delivery failures across SNS subscription protocols"
status: active
severity: HIGH
triggers:
  - "delivery failed"
  - "message not delivered"
  - "notification failed"
  - "NumberOfNotificationsFailed"
owner: devops-agent
objective: "Identify and resolve SNS message delivery failures"
context: "Delivery failures can occur for any subscription protocol (HTTP, Lambda, SQS, SMS, email, mobile push). Each protocol has different failure modes and retry policies. Delivery status logging provides specific error details."
---

## Phase 1 — Triage

MUST:
- Check delivery failure metrics: `aws cloudwatch get-metric-statistics --namespace AWS/SNS --metric-name NumberOfNotificationsFailed --dimensions Name=TopicName,Value=<topic-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Check delivery success metrics: `aws cloudwatch get-metric-statistics --namespace AWS/SNS --metric-name NumberOfNotificationsDelivered --dimensions Name=TopicName,Value=<topic-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Verify delivery status logging is enabled: `aws sns get-topic-attributes --topic-arn <topic-arn> --query 'Attributes.{HTTP:HTTPSuccessFeedbackRoleArn,Lambda:LambdaSuccessFeedbackRoleArn,SQS:SQSSuccessFeedbackRoleArn}'`
- If logging enabled, check logs: `aws logs filter-log-events --log-group-name sns/<region>/<account-id>/<topic-name> --filter-pattern "FAILURE"`

SHOULD:
- Check subscription status: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`
- Verify endpoint is reachable (HTTP) or exists (SQS, Lambda)
- Check for subscription confirmation status (PendingConfirmation)

MAY:
- Check CloudTrail for publish events
- Review DLQ for undeliverable messages

## Phase 2 — Remediate

MUST:
- If endpoint unreachable: fix the endpoint (HTTP URL, Lambda function, SQS queue)
- If permission denied: fix IAM permissions or resource policy
- If subscription pending: confirm the subscription
- Enable delivery status logging if not configured

SHOULD:
- Configure DLQ on subscriptions to capture failed messages
- Set up CloudWatch alarms on NumberOfNotificationsFailed
- Review and fix endpoint health

MAY:
- Implement retry logic in the subscriber
- Use SNS message filtering to reduce unnecessary deliveries

## Common Issues

- symptoms: "Messages published but not received by subscriber"
  diagnosis: "Subscription in PendingConfirmation state or filter policy excluding messages."
  resolution: "Confirm subscription. Check filter policy matches message attributes."

- symptoms: "HTTP endpoint returning 5xx errors"
  diagnosis: "Subscriber endpoint is unhealthy. SNS will retry per HTTP delivery policy."
  resolution: "Fix the endpoint. Messages will be retried for up to 23 days."

## Output Format

```yaml
root_cause: "delivery_failure — <specific_cause>"
evidence:
  - type: delivery_metrics
    content: "<failed/delivered counts>"
  - type: delivery_logs
    content: "<specific failure reason>"
severity: HIGH
mitigation:
  immediate: "Fix endpoint or subscription configuration"
  long_term: "Enable delivery logging and DLQ for all subscriptions"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Message payloads may contain sensitive application data
- Subscription endpoints reveal integration architecture
- SMS messages may contain customer PII or authentication codes
- Classification: CONFIDENTIAL - message content and subscriber endpoint data

## Prohibited Actions

- NEVER delete SNS topics with active subscriptions in production
- NEVER modify filter policies on production subscriptions without testing
- NEVER change access policies that could disrupt cross-account message delivery
- NEVER purge dead-letter queues without analyzing failed messages

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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

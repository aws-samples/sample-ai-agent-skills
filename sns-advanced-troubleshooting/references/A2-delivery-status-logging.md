---
title: "A2 — Delivery Status Logging Issues"
description: "Diagnose delivery status logging configuration problems"
status: active
severity: MEDIUM
triggers:
  - "delivery status logging"
  - "delivery logs"
  - "no logs"
  - "feedback role"
owner: devops-agent
objective: "Configure and troubleshoot SNS delivery status logging"
context: "Delivery status logging sends success/failure information to CloudWatch Logs. It must be enabled per protocol by setting feedback role ARNs on the topic. The IAM role must allow SNS to write to CloudWatch Logs."
---

## Phase 1 — Triage

MUST:
- Check topic attributes for logging config: `aws sns get-topic-attributes --topic-arn <topic-arn>`
- Look for *SuccessFeedbackRoleArn and *FailureFeedbackRoleArn attributes
- Verify the IAM role exists: `aws iam get-role --role-name <feedback-role>`
- Check role trust policy allows sns.amazonaws.com
- Check CloudWatch Logs for the log group: `aws logs describe-log-groups --log-group-name-prefix sns/`

SHOULD:
- Verify role has logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents permissions
- Check SuccessFeedbackSampleRate (0-100) — controls what percentage of successes are logged
- Verify log group exists in the correct Region

MAY:
- Check for log group retention settings
- Review CloudTrail for SetTopicAttributes calls

## Phase 2 — Remediate

MUST:
- If not configured: set feedback role ARNs on the topic for each protocol
- If role missing: create IAM role with sns.amazonaws.com trust and CloudWatch Logs permissions
- If logs not appearing: verify role permissions and trust policy

SHOULD:
- Enable logging for all protocols used by the topic
- Set SuccessFeedbackSampleRate to 100 for debugging, lower for production
- Configure log retention to manage costs

MAY:
- Set up CloudWatch Logs Insights queries for delivery analysis
- Implement automated alerting on delivery failures from logs

## Common Issues

- symptoms: "No delivery logs in CloudWatch"
  diagnosis: "Delivery status logging not enabled on the topic."
  resolution: "Set *SuccessFeedbackRoleArn and *FailureFeedbackRoleArn for each protocol."

- symptoms: "Feedback role exists but logs not appearing"
  diagnosis: "Role trust policy doesn't allow sns.amazonaws.com or missing CloudWatch Logs permissions."
  resolution: "Update trust policy and add logs:* permissions."

## Output Format

```yaml
root_cause: "delivery_logging — <specific_cause>"
evidence:
  - type: topic_attributes
    content: "<feedback role configuration>"
  - type: iam_role
    content: "<role trust and permissions>"
severity: MEDIUM
mitigation:
  immediate: "Configure delivery status logging"
  long_term: "Enable logging for all protocols and set up alerting"
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

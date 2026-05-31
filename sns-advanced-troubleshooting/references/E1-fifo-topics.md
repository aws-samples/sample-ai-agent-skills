---
title: "E1 — FIFO Topic Issues"
description: "Diagnose FIFO topic ordering and delivery problems"
status: active
severity: HIGH
triggers:
  - "FIFO topic"
  - "ordered delivery"
  - "message ordering"
  - "message group"
owner: devops-agent
objective: "Resolve FIFO topic configuration, ordering, and delivery issues"
context: "FIFO topics provide strict message ordering within message groups and exactly-once delivery. They require .fifo suffix and only support SQS FIFO queue subscriptions. Messages are ordered per MessageGroupId."
---

## Phase 1 — Triage

MUST:
- Check topic attributes: `aws sns get-topic-attributes --topic-arn <topic-arn>`
- Verify topic name ends with .fifo
- Check FifoTopic attribute is true
- List subscriptions: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`
- Verify all subscriptions are SQS FIFO queues

SHOULD:
- Check ContentBasedDeduplication setting
- Verify MessageGroupId is being set on published messages
- Check SQS FIFO queue configuration for subscribers

MAY:
- Review message ordering in the SQS FIFO queue
- Check for deduplication issues

## Phase 2 — Remediate

MUST:
- If not FIFO: create a new topic with .fifo suffix (cannot convert standard to FIFO)
- If non-FIFO subscription: use SQS FIFO queue (Lambda, HTTP not supported directly)
- If ordering wrong: verify MessageGroupId usage — ordering is per group

SHOULD:
- Use meaningful MessageGroupId values for logical ordering
- Enable ContentBasedDeduplication or provide MessageDeduplicationId
- Configure SQS FIFO queue with appropriate visibility timeout

MAY:
- Implement fan-out pattern: FIFO topic → multiple SQS FIFO queues → Lambda
- Monitor message throughput (300 msg/s per message group, 3000 with batching)

## Common Issues

- symptoms: "Cannot subscribe Lambda to FIFO topic"
  diagnosis: "FIFO topics only support SQS FIFO queue subscriptions."
  resolution: "Subscribe SQS FIFO queue to topic. Trigger Lambda from the SQS queue."

- symptoms: "Messages delivered out of order"
  diagnosis: "Different MessageGroupId values — ordering is per group, not global."
  resolution: "Use the same MessageGroupId for messages that must be ordered together."

## Output Format

```yaml
root_cause: "fifo_topic — <specific_cause>"
evidence:
  - type: topic_attributes
    content: "<FIFO topic configuration>"
  - type: subscriptions
    content: "<subscription list and types>"
severity: HIGH
mitigation:
  immediate: "Fix FIFO configuration or subscription type"
  long_term: "Implement proper message group strategy"
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

---
title: "E2 — Message Deduplication Issues"
description: "Diagnose message deduplication problems in FIFO topics"
status: active
severity: MEDIUM
triggers:
  - "deduplication"
  - "duplicate messages"
  - "MessageDeduplicationId"
  - "content-based deduplication"
owner: devops-agent
objective: "Resolve message deduplication configuration and behavior issues"
context: "FIFO topics support deduplication via MessageDeduplicationId or content-based deduplication. The deduplication window is 5 minutes. Messages with the same deduplication ID within the window are treated as duplicates."
---

## Phase 1 — Triage

MUST:
- Check topic deduplication setting: `aws sns get-topic-attributes --topic-arn <topic-arn> --query 'Attributes.ContentBasedDeduplication'`
- Verify MessageDeduplicationId is being set on published messages
- Check if duplicate messages are appearing in subscriber queues
- Verify the 5-minute deduplication window timing

SHOULD:
- Check if content-based deduplication is appropriate for the use case
- Verify deduplication ID uniqueness strategy
- Review message publishing patterns for unintentional duplicates

MAY:
- Analyze message patterns for deduplication effectiveness
- Check SQS FIFO queue deduplication settings (may also deduplicate)

## Phase 2 — Remediate

MUST:
- If duplicates appearing: verify deduplication is enabled (ContentBasedDeduplication or explicit ID)
- If messages being dropped: check if different messages have the same deduplication ID
- If content-based not working: ensure message bodies are truly identical for duplicates

SHOULD:
- Use explicit MessageDeduplicationId for precise control
- Use UUIDs or idempotency keys as deduplication IDs
- Understand the 5-minute deduplication window

MAY:
- Implement application-level deduplication for additional safety
- Monitor deduplication metrics

## Common Issues

- symptoms: "Duplicate messages in SQS FIFO queue"
  diagnosis: "ContentBasedDeduplication disabled and no MessageDeduplicationId provided."
  resolution: "Enable ContentBasedDeduplication or provide MessageDeduplicationId on publish."

- symptoms: "Unique messages being dropped as duplicates"
  diagnosis: "Same MessageDeduplicationId used for different messages within 5-minute window."
  resolution: "Use unique deduplication IDs for each distinct message."

## Output Format

```yaml
root_cause: "deduplication — <specific_cause>"
evidence:
  - type: dedup_config
    content: "<deduplication settings>"
  - type: message_pattern
    content: "<deduplication ID usage>"
severity: MEDIUM
mitigation:
  immediate: "Fix deduplication configuration"
  long_term: "Implement proper deduplication ID strategy"
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

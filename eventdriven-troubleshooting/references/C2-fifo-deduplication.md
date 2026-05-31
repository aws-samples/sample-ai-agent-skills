---
title: "C2 — FIFO Deduplication"
description: "Diagnose and resolve SQS FIFO queue message deduplication issues"
status: active
severity: MEDIUM
triggers:
  - "deduplication"
  - "duplicate message FIFO"
  - "MessageDeduplicationId"
  - "content-based deduplication"
  - "dedup window"
owner: devops-agent
objective: "Resolve FIFO deduplication issues to ensure exactly-once message delivery"
context: "SQS FIFO queues provide exactly-once processing using deduplication. Deduplication can be content-based (SHA-256 hash of message body) or explicit (MessageDeduplicationId). The deduplication window is 5 minutes — sending the same message within this window is deduplicated. Issues arise from incorrect deduplication IDs, content-based dedup with varying metadata, or messages sent outside the dedup window being treated as duplicates."
---

## Phase 1 — Triage

MUST:
- Check if content-based deduplication is enabled: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names ContentBasedDeduplication`
- Verify the producer is setting MessageDeduplicationId (if content-based is disabled)
- Check if messages are being rejected as duplicates (producer receives successful send but message doesn't appear)
- Review producer logs for SendMessage responses
- Check the deduplication scope: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names DeduplicationScope`

SHOULD:
- Verify the MessageDeduplicationId strategy — is it unique for distinct messages?
- Check if the same message is being sent multiple times within the 5-minute window
- Review if message attributes (not included in content-based hash) differ between "duplicate" messages

MAY:
- Analyze the producer's retry logic for duplicate sends
- Check if network issues are causing the producer to retry sends
- Review the deduplication ID generation logic

## Phase 2 — Remediate

MUST:
- If content-based dedup incorrectly deduplicates different messages: use explicit MessageDeduplicationId instead
- If MessageDeduplicationId is not unique: fix the ID generation to produce unique IDs for distinct messages
- If producer retries cause duplicates outside dedup window: implement idempotency in the consumer as a safety net
- If dedup is disabled and duplicates appear: enable content-based dedup or set MessageDeduplicationId

SHOULD:
- Use UUIDs or business-specific IDs for MessageDeduplicationId
- Implement producer-side deduplication tracking for critical messages
- Set DeduplicationScope to messageGroup for high throughput FIFO

MAY:
- Implement a deduplication check in the consumer as defense-in-depth
- Document the deduplication strategy for the application
- Set up monitoring for message send success rates

## Common Issues

- symptoms: "Messages silently dropped — producer gets success but consumer never receives"
  diagnosis: "Message deduplicated because the same MessageDeduplicationId was sent within 5 minutes."
  resolution: "Ensure each unique message has a unique deduplication ID. Check for accidental ID reuse."

- symptoms: "Content-based dedup treats different messages as duplicates"
  diagnosis: "Messages have the same body but different attributes — attributes are not included in the content hash."
  resolution: "Use explicit MessageDeduplicationId that includes attribute differences."

- symptoms: "Duplicate messages received despite FIFO queue"
  diagnosis: "Messages sent more than 5 minutes apart with the same content are not deduplicated."
  resolution: "This is expected behavior. Implement consumer-side idempotency for messages that may be resent."

- symptoms: "SendMessage fails with 'MessageDeduplicationId is required'"
  diagnosis: "Content-based deduplication is disabled and no MessageDeduplicationId was provided."
  resolution: "Either enable content-based deduplication or provide MessageDeduplicationId in every SendMessage call."

## Output Format

```yaml
root_cause: "FIFO deduplication issue — <specific_cause>"
evidence:
  - type: queue_config
    content: "<deduplication settings>"
  - type: producer_behavior
    content: "<MessageDeduplicationId usage pattern>"
severity: MEDIUM
mitigation:
  immediate: "Fix deduplication ID strategy or enable content-based dedup"
  long_term: "Implement robust deduplication strategy with consumer-side idempotency"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (get-queue-attributes): GREEN — read-only"
  - "Phase 2 deduplication setting changes: YELLOW — affects message delivery guarantees"
  - "Phase 2 producer code changes: YELLOW — affects message sending behavior"

## Escalation Conditions

escalation_conditions:
  - "Deduplication is causing message loss in production"
  - "Duplicate processing is causing data inconsistency"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Deduplication strategy change requires coordinated producer/consumer updates"
  - "Issue affects financial or audit-critical messages"

## Data Sensitivity

data_sensitivity:
  - command: "get-queue-attributes"
    sensitivity: LOW
    contains: "Queue configuration settings"
  - command: "SendMessage responses"
    sensitivity: MEDIUM
    contains: "Message IDs and deduplication IDs"
  - command: "message body hash"
    sensitivity: LOW
    contains: "SHA-256 hash of message content"

## Prohibited Actions

prohibited_actions:
  - "NEVER disable deduplication on a FIFO queue without implementing alternative dedup logic"
  - "NEVER use timestamps alone as MessageDeduplicationId (clock skew can cause issues)"
  - "NEVER assume the 5-minute dedup window protects against all duplicates"
  - "NEVER change deduplication strategy during active message processing without coordination"

## Phase 3 — Rollback

- Before changing deduplication settings, document the current configuration
- If content-based dedup was enabled/disabled, revert if it causes message loss or duplicates
- If producer deduplication ID logic was changed, roll back if it causes issues
- Coordinate producer and consumer changes to avoid inconsistency during rollback
- Monitor message delivery and duplicate rates after any changes

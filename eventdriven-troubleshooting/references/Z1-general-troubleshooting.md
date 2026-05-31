---
title: "Z1 — General Event-Driven Troubleshooting"
description: "General troubleshooting guide for SQS, SNS, and EventBridge issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "SQS error"
  - "SNS error"
  - "EventBridge error"
  - "event-driven troubleshooting"
  - "messaging issue"
owner: devops-agent
objective: "Provide a general troubleshooting framework for event-driven service issues and route to specific runbooks"
context: "This runbook covers general troubleshooting for SQS, SNS, and EventBridge when the issue does not clearly map to a specific runbook. It provides a systematic diagnostic framework covering message production, delivery, processing, and integration. Use this as a starting point and pivot to specific runbooks once the issue category is identified."
---

## Phase 1 — Triage

MUST:
- Identify the affected service(s): SQS, SNS, EventBridge, or a combination
- Check service health: review AWS Health Dashboard for SQS, SNS, EventBridge
- For SQS: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names All`
- For SNS: `aws sns get-topic-attributes --topic-arn <topic-arn>`
- For EventBridge: `aws events describe-event-bus --name <bus-name>`
- Check CloudWatch metrics for the affected resource (message counts, errors, latency)
- Review CloudTrail for recent API errors on the affected service

SHOULD:
- Verify IAM permissions for the calling principal
- Check if the resource is encrypted and KMS key is accessible
- Review recent configuration changes via CloudTrail
- Verify network connectivity (VPC endpoints if applicable)

MAY:
- Check if the issue correlates with recent deployments or infrastructure changes
- Review AWS service limits and quotas
- Check if the issue affects multiple resources or is isolated

## Phase 2 — Remediate

MUST:
- Route to the appropriate specific runbook based on triage findings:
  - Message delay/delivery → A2 (Message Delay)
  - DLQ issues → B1 (DLQ Messages)
  - Visibility timeout → B2 (Visibility Timeout)
  - FIFO ordering → C1 (FIFO Ordering)
  - FIFO deduplication → C2 (FIFO Deduplication)
  - SNS delivery → D1 (SNS Delivery Failures)
  - SNS filtering → D2 (SNS Filtering)
  - EventBridge rules → E1 (EventBridge Rule Matching)
  - EventBridge targets → E2 (EventBridge Target Failures)
  - Cross-account → F1 (Cross-Account Delivery)
  - Encryption → F2 (Encryption Issues)
  - Subscription → G1 (Subscription Confirmation)
- If the issue spans multiple categories, address the most critical component first

SHOULD:
- Document the troubleshooting steps and findings
- Verify the fix resolves the root cause, not just the symptom
- Check for related issues across connected services

MAY:
- Create a post-incident report for significant outages
- Update monitoring and alerting based on the issue discovered
- Propose architectural improvements to prevent recurrence

## Common Issues

- symptoms: "Messages not flowing through the pipeline"
  diagnosis: "Could be producer, queue/topic, or consumer issue. Check each component."
  resolution: "Verify producer is sending, queue/topic is receiving, and consumer is processing."

- symptoms: "Intermittent message delivery failures"
  diagnosis: "Transient errors, throttling, or resource limits."
  resolution: "Check CloudWatch for throttling metrics. Review service quotas. Implement retries."

- symptoms: "Event-driven architecture latency spike"
  diagnosis: "Queue depth increasing, consumer bottleneck, or downstream dependency slow."
  resolution: "Scale consumers. Check downstream dependencies. Review queue metrics."

- symptoms: "Service returns AccessDeniedException"
  diagnosis: "IAM policy, resource policy, or KMS key policy missing required permissions."
  resolution: "Review and fix the relevant policy. Check for SCP restrictions."

## Output Format

```yaml
root_cause: "Event-driven issue — <specific_cause>"
evidence:
  - type: service_status
    content: "<affected service configuration and metrics>"
  - type: error_details
    content: "<specific error messages or CloudTrail events>"
severity: MEDIUM
mitigation:
  immediate: "<specific remediation based on root cause>"
  long_term: "Implement comprehensive event-driven monitoring and alerting"
routed_to: "<specific runbook ID if applicable>"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (get/describe/list): GREEN — read-only"
  - "Phase 2 remediation varies by specific runbook: YELLOW to RED"
  - "Refer to the specific runbook's safety ratings for detailed guidance"

## Escalation Conditions

escalation_conditions:
  - "Issue cannot be categorized into a specific runbook after triage"
  - "Multiple event-driven services are failing simultaneously"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Issue involves potential data loss or message corruption"
  - "AWS service-side issue suspected"

## Data Sensitivity

data_sensitivity:
  - command: "get-queue-attributes / get-topic-attributes"
    sensitivity: MEDIUM
    contains: "Service configuration and policies"
  - command: "receive-message / message content"
    sensitivity: HIGH
    contains: "Message body with potential business data"
  - command: "CloudTrail events"
    sensitivity: HIGH
    contains: "API call details including principal ARNs"

## Prohibited Actions

prohibited_actions:
  - "NEVER purge production queues without explicit approval"
  - "NEVER delete topics or event buses during troubleshooting"
  - "NEVER read production message content without data handling authorization"
  - "NEVER modify resource policies to allow unrestricted access"

## Phase 3 — Rollback

- Document the pre-change state before any remediation
- Follow the rollback procedures in the specific runbook used for remediation
- If multiple changes were made, roll back in reverse order
- Verify message flow and processing resume normally after rollback
- Retain logs and metrics for post-incident analysis

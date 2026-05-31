---
title: "A3 — Subscription Filter Issues"
description: "Diagnose subscription filter delivery failures"
status: active
severity: HIGH
triggers:
  - "subscription filter"
  - "logs not delivered"
  - "Kinesis delivery"
  - "Lambda subscription"
  - "log destination"
  - "LimitExceededException"
owner: devops-agent
objective: "Identify why a subscription filter is not delivering log events to its destination"
context: "Subscription filters stream log events to Lambda, Kinesis Data Streams, Kinesis Data Firehose, or another account's destination. Each log group supports a maximum of 2 subscription filters. Failures are caused by IAM permissions, destination throttling, filter pattern mismatch, or the 2-filter limit."
---

## Phase 1 — Triage

MUST:
- List subscription filters: `aws logs describe-subscription-filters --log-group-name <name>`
- Verify the destination exists and is active (Lambda function, Kinesis stream, Firehose)
- Check IAM permissions: the log group needs permission to write to the destination
- Verify the filter pattern matches expected log events: `aws logs test-metric-filter --filter-pattern <pattern> --log-event-messages '<test-message>'`

SHOULD:
- Check destination throttling (Kinesis shard limits, Lambda concurrency)
- Verify cross-account destination policy if delivering to another account
- Check CloudTrail for subscription filter API errors

MAY:
- Test with a simple filter pattern ("" matches all events) to isolate pattern issues
- Check destination CloudWatch metrics for errors

## Phase 2 — Remediate

MUST:
- Fix IAM permissions on the destination resource policy
- Fix filter pattern if it doesn't match expected events
- Remove a filter if at the 2-filter limit before adding a new one

SHOULD:
- Increase Kinesis shard count if throttled
- Increase Lambda concurrency limit if throttled
- Use Kinesis Data Firehose for fan-out to multiple destinations

MAY:
- Set up a dead-letter queue for failed deliveries
- Monitor subscription filter metrics: ForwardedLogEvents, DeliveryErrors, DeliveryThrottling

## Common Issues

- symptoms: "LimitExceededException when creating subscription filter"
  diagnosis: "Log group already has 2 subscription filters (maximum)."
  resolution: "Remove an existing filter or use Kinesis Data Firehose to fan out."

- symptoms: "Subscription filter exists but no events delivered"
  diagnosis: "Filter pattern does not match any log events."
  resolution: "Test the filter pattern with test-metric-filter. Use empty string to match all."

- symptoms: "Intermittent delivery failures"
  diagnosis: "Destination is being throttled (Kinesis shard limit or Lambda concurrency)."
  resolution: "Scale the destination: add Kinesis shards or increase Lambda concurrency."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix IAM permissions on destination | YELLOW | Resource policy changes |
| Fix filter pattern | GREEN | Configuration change |
| Remove filter (at 2-filter limit) | YELLOW | Stops log delivery to that destination |
| Increase Kinesis shard count | YELLOW | Cost increase; affects stream throughput |

## Escalation Conditions

- Subscription filter delivering to security monitoring pipeline broken — escalate to security operations
- Lambda subscription throttling affecting real-time processing — escalate to application team
- Cross-account subscription filter issues — escalate to destination account owner
- Filter limit reached (2 per log group) — escalate to architecture team for fan-out design

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws logs describe-subscription-filters` | MEDIUM | Exposes filter configuration and destination ARNs |
| `aws logs test-metric-filter` | LOW | Tests pattern matching only |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** remove subscription filters feeding security monitoring without approval

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Fixed filter pattern | Restore previous filter pattern |
| Removed subscription filter | Recreate the filter with original configuration |
| Changed destination permissions | Restore previous resource policy |
| Increased Kinesis shards | Reduce shard count (use UpdateShardCount) |

## Output Format

```yaml
root_cause: "subscription_filter — <specific_cause>"
evidence:
  - type: subscription_filters
    content: "<filter configuration>"
  - type: destination_status
    content: "<destination health>"
severity: HIGH
mitigation:
  immediate: "Fix filter pattern, IAM permissions, or destination capacity"
  long_term: "Monitor DeliveryErrors and DeliveryThrottling metrics"
```

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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"

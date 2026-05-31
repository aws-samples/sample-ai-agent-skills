---
title: "G1 — SQS Event Source Mapping Issues"
description: "Diagnose SQS → Lambda event source mapping failures"
status: active
severity: HIGH
triggers:
  - "event source mapping.*SQS"
  - "messages.*not processed"
  - "SQS.*Lambda.*error"
owner: devops-agent
objective: "Fix SQS event source mapping to process messages reliably"
context: "Lambda polls SQS using long polling. Batch size up to 10 (standard) or 10000 (FIFO). Failed messages return to queue after visibility timeout. DLQ on the SQS queue (not Lambda) handles poison messages. Lambda deletes messages only on successful processing."
---

## Common Issues

- symptoms: "Messages stuck in queue, not being processed"
  diagnosis: "Event source mapping disabled, Lambda throttled, or execution role missing sqs:ReceiveMessage."
  resolution: "Check ESM state. Check Lambda throttles. Add sqs:ReceiveMessage, sqs:DeleteMessage, sqs:GetQueueAttributes to execution role."

- symptoms: "Messages processed but reappear (duplicates)"
  diagnosis: "Function fails or times out. Message returns to queue after visibility timeout."
  resolution: "Increase visibility timeout to 6x Lambda timeout. Fix function errors. Use DLQ for poison messages."

- symptoms: "FIFO queue messages processed out of order"
  diagnosis: "Lambda processes multiple batches concurrently (up to 10 per message group)."
  resolution: "Set MaximumConcurrency on ESM to 1 for strict ordering. Or design for idempotency."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
```

## Escalation Conditions

- Function serves a production API or critical workload
- Fix requires changing reserved concurrency settings
- Function is processing sensitive data (PII, financial, healthcare)
- Resolution involves modifying VPC configuration or security groups
- Multiple functions affected suggesting account-level issue

## Data Sensitivity

HIGH - Lambda function code and environment variables may contain secrets, API keys, database credentials, and encryption keys. CloudWatch logs may capture sensitive request/response data. X-Ray traces may contain PII in segment metadata.

## Prohibited Actions

- NEVER suggest setting reserved concurrency to 0 - this effectively disables the function
- NEVER suggest deleting a function alias that is serving live traffic
- NEVER recommend removing or replacing the execution role on a running function without verifying the new role has equivalent permissions
- NEVER suggest publishing function code changes directly to a production alias without testing
- NEVER expose environment variable values in logs or diagnostic output - they may contain secrets

## Phase 3 - Rollback

1. If function configuration was changed, revert using: `aws lambda update-function-configuration --function-name <name> --memory-size <original> --timeout <original>`
2. If reserved concurrency was modified, restore original value: `aws lambda put-function-concurrency --function-name <name> --reserved-concurrent-executions <original>`
3. If execution role was modified, revert IAM policy changes and verify function can still access required resources
4. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<esm_disabled|permission|visibility_timeout|throttling|fifo_ordering>"
severity: HIGH
mitigation:
  immediate: "Fix ESM state, permissions, or visibility timeout"
  long_term: "Configure DLQ, implement idempotency, set appropriate batch size"
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
  - command: "get-function-configuration"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "invoke"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest Resource: * in Lambda execution role"
  - "NEVER suggest disabling VPC configuration to fix connectivity"
  - "NEVER expose function URLs without authentication"

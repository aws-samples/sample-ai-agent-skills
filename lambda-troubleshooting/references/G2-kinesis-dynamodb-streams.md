---
title: "G2 — Kinesis / DynamoDB Streams Event Source Issues"
description: "Diagnose stream-based event source mapping failures"
status: active
severity: HIGH
triggers:
  - "IteratorAge"
  - "stream.*behind"
  - "bisect.*batch"
owner: devops-agent
objective: "Fix stream processing to keep up with incoming records"
context: "Lambda reads from shards in parallel. One Lambda invocation per shard (or per batch with parallelization factor). Failed batches retry until record expires. IteratorAge metric shows how far behind processing is."
---

## Common Issues

- symptoms: "IteratorAge increasing (falling behind)"
  diagnosis: "Processing slower than incoming rate. Or errors causing retries that block the shard."
  resolution: "Increase parallelization factor (up to 10). Enable bisect-on-error. Increase batch size. Optimize function."

- symptoms: "One bad record blocks entire shard"
  diagnosis: "Poison record causes repeated failures. Lambda retries until record expires."
  resolution: "Enable BisectBatchOnFunctionError and MaximumRetryAttempts. Configure OnFailure destination."

- symptoms: "No records being processed"
  diagnosis: "ESM disabled, stream not active, or execution role missing stream permissions."
  resolution: "Check ESM state. Verify stream is ACTIVE. Add dynamodb:GetRecords, dynamodb:GetShardIterator, dynamodb:DescribeStream, dynamodb:ListStreams (or kinesis: equivalents)."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
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

1. If execution role was modified, revert IAM policy changes and verify function can still access required resources
2. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<falling_behind|poison_record|esm_disabled|permission>"
severity: HIGH
mitigation:
  immediate: "Enable bisect-on-error, increase parallelization"
  long_term: "Implement error handling, configure failure destinations, monitor IteratorAge"
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

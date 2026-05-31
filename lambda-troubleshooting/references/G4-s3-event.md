---
title: "G4 — S3 Event Notification Issues"
description: "Diagnose S3 event notification to Lambda failures"
status: active
severity: MEDIUM
triggers:
  - "S3.*not triggering"
  - "event notification.*Lambda"
owner: devops-agent
objective: "Fix S3 event notifications to Lambda"
context: "S3 event notifications invoke Lambda asynchronously. Requires: resource policy on Lambda, event notification configured on bucket, and correct prefix/suffix filters. S3 delivers at-least-once (duplicates possible)."
---

## Common Issues

- symptoms: "S3 upload doesn't trigger Lambda"
  diagnosis: "Missing resource policy, event notification not configured, or prefix/suffix filter doesn't match."
  resolution: "Check bucket notification config. Add Lambda resource policy for s3.amazonaws.com. Verify filters."

- symptoms: "Lambda triggered but receives unexpected event format"
  diagnosis: "Function expects different event structure. S3 events have specific JSON format."
  resolution: "Parse event['Records'][0]['s3']['bucket']['name'] and event['Records'][0]['s3']['object']['key']."

- symptoms: "Recursive invocation (Lambda writes to same bucket that triggers it)"
  diagnosis: "Lambda writes output to the same bucket/prefix that triggers it, causing infinite loop."
  resolution: "Use different prefix/suffix for output. Or use a different bucket. Enable recursive loop detection."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
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
root_cause: "<resource_policy|notification_config|filter_mismatch|recursive_loop>"
severity: MEDIUM
mitigation:
  immediate: "Fix notification config and resource policy"
  long_term: "Use separate input/output prefixes, enable recursive loop detection"
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

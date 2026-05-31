---
title: "C3 — Account Concurrency Limit"
description: "Diagnose account-level Lambda concurrency limit issues"
status: active
severity: HIGH
triggers:
  - "account.*concurrency.*limit"
  - "ConcurrentExecutions.*1000"
owner: devops-agent
objective: "Manage account-level concurrency to prevent widespread throttling"
context: "Default account limit is 1000 concurrent executions per region. This is shared across ALL functions. Can be increased via Service Quotas. Burst limit is 500-3000 immediate, then +500/minute."
---

## Common Issues

- symptoms: "Multiple functions throttled simultaneously"
  diagnosis: "Account-level concurrency limit reached."
  resolution: "Request increase via Service Quotas. Set reserved concurrency on critical functions."

- symptoms: "Concurrency limit recently increased but still throttling"
  diagnosis: "Burst limit still applies. Scaling is gradual after initial burst."
  resolution: "Use provisioned concurrency for immediate capacity. Or use SQS to buffer requests."


## Safety Ratings

```
safety_ratings:
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
  - "Diagnostic investigation: GREEN - No changes to infrastructure"
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

1. If reserved concurrency was modified, restore original value: `aws lambda put-function-concurrency --function-name <name> --reserved-concurrent-executions <original>`
2. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<account_limit_reached|burst_limit|gradual_scaling>"
severity: HIGH
mitigation:
  immediate: "Request limit increase, set reserved concurrency on critical functions"
  long_term: "Implement concurrency budgeting, use async patterns with SQS"
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

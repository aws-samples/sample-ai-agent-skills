---
title: "C1 — Lambda Throttling"
description: "Diagnose Lambda invocation throttling"
status: active
severity: HIGH
triggers:
  - "Rate exceeded"
  - "TooManyRequestsException"
  - "Throttles.*metric"
  - "429"
owner: devops-agent
objective: "Identify the throttling source and restore normal invocation rates"
context: "Lambda throttling occurs when concurrent executions exceed limits. Account default: 1000 concurrent per region. Throttled sync invocations get 429. Throttled async invocations are retried then sent to DLQ. Reserved concurrency can also cause throttling."
---

## Phase 1 — Triage

MUST:
- Check Throttles metric: `aws cloudwatch get-metric-statistics --metric-name Throttles`
- Check ConcurrentExecutions metric at account and function level
- Check reserved concurrency: `aws lambda get-function-concurrency --function-name <name>`
- Check account limits: `aws lambda get-account-settings` → ConcurrentExecutions

SHOULD:
- Check if reserved concurrency is set too low (caps this function)
- Check if OTHER functions are consuming all unreserved concurrency
- Check burst concurrency limit (500-3000 depending on region, then +500/minute)

## Common Issues

- symptoms: "Throttles > 0, no reserved concurrency set"
  diagnosis: "Account-level concurrent execution limit reached."
  resolution: "Request limit increase via Service Quotas. Or set reserved concurrency on less critical functions to protect this one."

- symptoms: "Throttles > 0, reserved concurrency is set to low value"
  diagnosis: "Reserved concurrency is a CAP. Function can't exceed it."
  resolution: "Increase reserved concurrency. Remember: reserved units are removed from the shared pool."

- symptoms: "Burst of throttles at start, then resolves"
  diagnosis: "Burst concurrency limit hit. Lambda scales at 500-3000 immediate + 500/minute after."
  resolution: "Use provisioned concurrency for predictable bursts. Or implement client-side retry with backoff."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
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
root_cause: "<account_limit|reserved_too_low|burst_limit|other_functions_consuming>"
evidence:
  - type: cloudwatch_metric
    content: "<Throttles and ConcurrentExecutions data>"
severity: HIGH
mitigation:
  immediate: "Increase reserved concurrency or request account limit increase"
  long_term: "Implement concurrency management strategy, use SQS to buffer"
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

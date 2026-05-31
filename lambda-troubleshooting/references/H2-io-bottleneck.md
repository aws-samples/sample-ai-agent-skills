---
title: "H2 — I/O Bottleneck"
description: "Diagnose Lambda performance issues caused by I/O operations"
status: active
severity: MEDIUM
triggers:
  - "slow.*API call"
  - "latency.*downstream"
  - "connection.*pool"
owner: devops-agent
objective: "Optimize I/O patterns for Lambda"
context: "Lambda functions often spend most time waiting on I/O: API calls, DB queries, S3 operations. Optimizations: connection reuse (keep-alive), parallel I/O (Promise.all/asyncio.gather), SDK client initialization outside handler."
---

## Common Issues

- symptoms: "Function duration dominated by API calls"
  diagnosis: "Sequential API calls. Each call adds latency."
  resolution: "Parallelize independent calls. Use Promise.all (Node.js) or asyncio.gather (Python)."

- symptoms: "First invocation slow, subsequent fast"
  diagnosis: "SDK client initialization in handler. Should be in global scope for reuse."
  resolution: "Move SDK client creation to global scope. Connections persist across warm invocations."

- symptoms: "Intermittent slow invocations"
  diagnosis: "Connection pool exhaustion or DNS resolution delays."
  resolution: "Configure SDK connection pool size. Enable keep-alive. Use VPC endpoints for AWS services."


## Safety Ratings

```
safety_ratings:
  - "Network configuration changes: YELLOW - May affect connectivity"
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

1. If VPC configuration was changed, restore original VPC settings: `aws lambda update-function-configuration --function-name <name> --vpc-config SubnetIds=<original>,SecurityGroupIds=<original>`
2. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<sequential_io|sdk_init_in_handler|connection_pool|dns>"
severity: MEDIUM
mitigation:
  immediate: "Parallelize I/O, move SDK init to global scope"
  long_term: "Use connection pooling, VPC endpoints, caching"
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

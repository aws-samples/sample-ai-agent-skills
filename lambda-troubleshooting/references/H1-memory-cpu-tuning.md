---
title: "H1 — Memory/CPU Tuning"
description: "Optimize Lambda memory and CPU allocation for performance and cost"
status: active
severity: MEDIUM
triggers:
  - "slow.*Lambda"
  - "optimize.*memory"
  - "CPU.*bound"
owner: devops-agent
objective: "Find optimal memory setting for performance and cost"
context: "Lambda memory controls CPU proportionally. 128 MB = ~0.07 vCPU. 1769 MB = 1 vCPU. 10240 MB = 6 vCPUs. More memory can make functions faster AND cheaper (less duration billed). Use Lambda Power Tuning to find the sweet spot."
---

## Common Issues

- symptoms: "Function is slow, memory usage is low"
  diagnosis: "CPU-bound workload with insufficient CPU. Memory is fine but CPU is fractional."
  resolution: "Increase memory to get more CPU. 1769 MB = 1 full vCPU. Test with Power Tuning."

- symptoms: "Function uses 90%+ memory but runs fast"
  diagnosis: "Memory-bound. At risk of OOM. Increase memory for safety margin."
  resolution: "Increase memory by 50-100% for headroom."


## Safety Ratings

```
safety_ratings:
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

1. If function configuration was changed, revert using: `aws lambda update-function-configuration --function-name <name> --memory-size <original> --timeout <original>`
2. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<cpu_starved|memory_bound|over_provisioned>"
severity: MEDIUM
mitigation:
  immediate: "Adjust memory based on workload profile"
  long_term: "Use Lambda Power Tuning tool for data-driven optimization"
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

---
title: "B3 — Provisioned Concurrency Issues"
description: "Diagnose provisioned concurrency configuration and scaling issues"
status: active
severity: MEDIUM
triggers:
  - "provisioned concurrency.*not ready"
  - "ProvisionedConcurrencySpillover"
  - "provisioned.*failed"
owner: devops-agent
objective: "Ensure provisioned concurrency is correctly configured and scaling"
context: "Provisioned concurrency pre-initializes execution environments. It eliminates cold starts but costs money when idle. It's set on a function version or alias, NOT on $LATEST. Auto-scaling can adjust PC based on utilization."
---

## Common Issues

- symptoms: "Cold starts still happening despite provisioned concurrency"
  diagnosis: "Traffic exceeds provisioned amount. Spillover invocations get cold starts."
  resolution: "Increase provisioned concurrency or add Application Auto Scaling."

- symptoms: "Provisioned concurrency set on $LATEST"
  diagnosis: "PC cannot be configured on $LATEST. Must use a published version or alias."
  resolution: "Publish a version, create an alias, set PC on the alias."

- symptoms: "ProvisionedConcurrencySpilloverInvocations metric > 0"
  diagnosis: "Demand exceeds provisioned amount. Spillover uses on-demand (cold starts possible)."
  resolution: "Increase PC or configure auto-scaling with target tracking on utilization."


## Safety Ratings

```
safety_ratings:
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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

1. If function configuration was changed, revert using: `aws lambda update-function-configuration --function-name <name> --memory-size <original> --timeout <original>`
2. If reserved concurrency was modified, restore original value: `aws lambda put-function-concurrency --function-name <name> --reserved-concurrent-executions <original>`
3. If a new version was published, point the alias back to the previous version: `aws lambda update-alias --function-name <name> --name <alias> --function-version <previous>`
4. If environment variables were changed, restore previous values: `aws lambda update-function-configuration --function-name <name> --environment Variables=<original>`
5. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<spillover|configured_on_latest|insufficient_pc|scaling_lag>"
severity: MEDIUM
mitigation:
  immediate: "Increase provisioned concurrency"
  long_term: "Configure auto-scaling on PC utilization metric"
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

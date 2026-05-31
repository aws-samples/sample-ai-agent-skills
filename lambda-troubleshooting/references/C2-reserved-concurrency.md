---
title: "C2 — Reserved Concurrency Misconfiguration"
description: "Diagnose issues caused by reserved concurrency settings"
status: active
severity: MEDIUM
triggers:
  - "reserved concurrency.*throttle"
  - "concurrency.*0"
owner: devops-agent
objective: "Fix reserved concurrency configuration"
context: "Reserved concurrency dedicates a portion of the account limit to a function. Setting it to 0 effectively disables the function. Setting it too low causes throttling. Setting it too high starves other functions."
---

## Common Issues

- symptoms: "Function never executes, reserved concurrency = 0"
  diagnosis: "Reserved concurrency of 0 disables the function entirely."
  resolution: "Remove reserved concurrency or set to appropriate value."

- symptoms: "Function throttled, reserved concurrency = 5, traffic is higher"
  diagnosis: "Reserved concurrency caps at 5 concurrent executions."
  resolution: "Increase to match expected peak concurrency."

- symptoms: "Other functions throttled after setting high reserved concurrency"
  diagnosis: "Reserved units are removed from the shared pool. 900 reserved = only 100 left for all other functions."
  resolution: "Balance reserved concurrency across functions. Don't over-reserve."


## Safety Ratings

```
safety_ratings:
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

1. If function configuration was changed, revert using: `aws lambda update-function-configuration --function-name <name> --memory-size <original> --timeout <original>`
2. If reserved concurrency was modified, restore original value: `aws lambda put-function-concurrency --function-name <name> --reserved-concurrent-executions <original>`
3. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<set_to_zero|too_low|starving_others>"
severity: MEDIUM
mitigation:
  immediate: "Adjust reserved concurrency value"
  long_term: "Document concurrency budget across all functions"
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

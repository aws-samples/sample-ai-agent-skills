---
title: "E3 — KMS Permission Issues"
description: "Diagnose KMS key access failures for Lambda environment variable encryption"
status: active
severity: MEDIUM
triggers:
  - "KMS.*AccessDenied"
  - "cannot decrypt.*environment"
owner: devops-agent
objective: "Restore KMS access for encrypted environment variables"
context: "Lambda encrypts environment variables with KMS. Default uses aws/lambda key. Custom CMK requires execution role to have kms:Decrypt. If KMS access is lost, function fails at init."
---

## Common Issues

- symptoms: "Function fails at init with KMS AccessDeniedException"
  diagnosis: "Execution role lacks kms:Decrypt on the custom CMK used for env var encryption."
  resolution: "Add kms:Decrypt permission for the CMK to the execution role."


## Safety Ratings

```
safety_ratings:
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
2. If environment variables were changed, restore previous values: `aws lambda update-function-configuration --function-name <name> --environment Variables=<original>`
3. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<missing_kms_decrypt|key_disabled|key_policy>"
severity: MEDIUM
mitigation:
  immediate: "Add kms:Decrypt to execution role or switch to default key"
  long_term: "Use Secrets Manager instead of encrypted env vars for sensitive data"
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

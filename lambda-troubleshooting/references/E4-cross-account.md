---
title: "E4 — Cross-Account Access Issues"
description: "Diagnose cross-account Lambda invocation and resource access failures"
status: active
severity: MEDIUM
triggers:
  - "cross-account"
  - "AssumeRole.*denied"
  - "not authorized.*account"
owner: devops-agent
objective: "Fix cross-account permissions for Lambda"
context: "Cross-account Lambda access requires: resource policy on the function (for invocation) AND/OR execution role with cross-account assume role (for accessing resources in other accounts). Both sides must grant access."
---

## Common Issues

- symptoms: "Cannot invoke Lambda from another account"
  diagnosis: "Resource policy doesn't include the other account's principal."
  resolution: "Add resource policy permission for the cross-account principal."

- symptoms: "Lambda can't access S3/DynamoDB in another account"
  diagnosis: "Execution role needs to assume a role in the target account, or target resource policy must allow cross-account access."
  resolution: "Use STS AssumeRole to get credentials for the target account, or add cross-account access in the resource policy."


## Safety Ratings

```
safety_ratings:
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
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

1. If execution role was modified, revert IAM policy changes and verify function can still access required resources
2. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<resource_policy|assume_role|target_resource_policy>"
severity: MEDIUM
mitigation:
  immediate: "Add cross-account permissions on both sides"
  long_term: "Use AWS Organizations SCPs to govern cross-account access"
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

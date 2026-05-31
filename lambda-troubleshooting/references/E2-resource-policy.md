---
title: "E2 — Resource Policy Issues"
description: "Diagnose Lambda resource-based policy issues preventing invocation"
status: active
severity: MEDIUM
triggers:
  - "is not authorized to invoke"
  - "resource policy"
  - "AddPermission"
owner: devops-agent
objective: "Fix resource policy to allow authorized invocations"
context: "Resource policies control WHO can invoke the function. Required for: API Gateway, S3 event notifications, CloudWatch Events, SNS, other AWS services, and cross-account invocations. Separate from the execution role."
---

## Common Issues

- symptoms: "API Gateway returns 500, Lambda logs show no invocation"
  diagnosis: "Resource policy doesn't allow API Gateway to invoke the function."
  resolution: "`aws lambda add-permission --function-name <name> --statement-id apigateway --action lambda:InvokeFunction --principal apigateway.amazonaws.com --source-arn <api-arn>`"

- symptoms: "S3 event notification not triggering Lambda"
  diagnosis: "Resource policy doesn't allow S3 to invoke the function."
  resolution: "Add permission for s3.amazonaws.com principal with source ARN of the bucket."

- symptoms: "Cross-account invocation fails"
  diagnosis: "Resource policy doesn't allow the other account's principal."
  resolution: "Add permission for the cross-account role/user ARN."


## Safety Ratings

```
safety_ratings:
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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

1. If a new version was published, point the alias back to the previous version: `aws lambda update-alias --function-name <name> --name <alias> --function-version <previous>`
2. If execution role was modified, revert IAM policy changes and verify function can still access required resources
3. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<missing_permission|wrong_principal|wrong_source_arn>"
severity: MEDIUM
mitigation:
  immediate: "Add resource policy permission for the invoking service"
  long_term: "Use IaC to manage resource policies alongside function deployment"
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

---
title: "G3 — API Gateway → Lambda Integration Issues"
description: "Diagnose API Gateway to Lambda integration failures"
status: active
severity: MEDIUM
triggers:
  - "Internal server error"
  - "502.*Bad Gateway"
  - "Execution failed due to configuration error"
owner: devops-agent
objective: "Fix API Gateway to Lambda integration"
context: "API Gateway invokes Lambda synchronously. Common issues: resource policy missing, timeout mismatch (APIGW max 29s, Lambda max 900s), response format wrong (proxy integration requires specific JSON), or Lambda error not handled."
---

## Common Issues

- symptoms: "502 Bad Gateway from API Gateway"
  diagnosis: "Lambda returned invalid response format for proxy integration. Must return {statusCode, headers, body}."
  resolution: "Fix Lambda response format. For proxy integration: return {statusCode: 200, body: JSON.stringify(data)}."

- symptoms: "500 Internal Server Error, Lambda never invoked"
  diagnosis: "Resource policy doesn't allow API Gateway to invoke Lambda."
  resolution: "Add Lambda resource policy for apigateway.amazonaws.com."

- symptoms: "API Gateway returns 504 timeout"
  diagnosis: "API Gateway timeout is 29 seconds max. Lambda function takes longer."
  resolution: "Optimize Lambda to complete within 29s. Or use async pattern (API GW → SQS → Lambda)."


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

1. If function configuration was changed, revert using: `aws lambda update-function-configuration --function-name <name> --memory-size <original> --timeout <original>`
2. If execution role was modified, revert IAM policy changes and verify function can still access required resources
3. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<response_format|resource_policy|timeout_mismatch|integration_config>"
severity: MEDIUM
mitigation:
  immediate: "Fix response format or resource policy"
  long_term: "Use async patterns for long-running operations"
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

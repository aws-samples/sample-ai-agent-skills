---
title: "A3 — Unhandled Exception / Application Error"
description: "Diagnose Lambda invocation failures caused by unhandled exceptions in function code"
status: active
severity: MEDIUM
triggers:
  - "Unhandled"
  - "errorMessage"
  - "errorType"
  - "Runtime.HandlerNotFound"
  - "module.*not found"
owner: devops-agent
objective: "Identify the exception and fix the function code or configuration"
context: "Unhandled exceptions return error responses to synchronous callers and trigger retries for async invocations. Common causes: missing modules, wrong handler path, null references, API errors, or missing environment variables."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch logs for the full stack trace
- Check function handler setting: `aws lambda get-function-configuration` → Handler
- Verify handler format: `<file>.<function>` (Python), `<file>.<export>` (Node.js), `<package>::<class>::<method>` (Java/.NET)
- Check for "Runtime.HandlerNotFound" or "module not found" errors

SHOULD:
- Check environment variables are set correctly
- Verify all dependencies are included in the deployment package
- Check if the error is intermittent (downstream service) or consistent (code bug)

## Common Issues

- symptoms: "Runtime.HandlerNotFound"
  diagnosis: "Handler path doesn't match the actual file/function in the deployment package."
  resolution: "Fix handler setting. Verify file name and exported function name match."

- symptoms: "Cannot find module '<package>'"
  diagnosis: "Dependency not included in deployment package or layer."
  resolution: "Include the dependency in the package. For Node.js: npm install in the package directory."

- symptoms: "KeyError or AttributeError on environment variable"
  diagnosis: "Environment variable not set or misspelled."
  resolution: "Check function environment variables. Use os.environ.get() with defaults."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
2. If a new version was published, point the alias back to the previous version: `aws lambda update-alias --function-name <name> --name <alias> --function-version <previous>`
3. If layers were changed, revert to previous layer versions: `aws lambda update-function-configuration --function-name <name> --layers <previous-layer-arns>`
4. If environment variables were changed, restore previous values: `aws lambda update-function-configuration --function-name <name> --environment Variables=<original>`
5. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<handler_not_found|missing_module|env_var|null_reference|downstream_error>"
evidence:
  - type: cloudwatch_logs
    content: "<stack trace>"
severity: MEDIUM
mitigation:
  immediate: "Fix the code or configuration"
  long_term: "Add error handling, input validation, integration tests"
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

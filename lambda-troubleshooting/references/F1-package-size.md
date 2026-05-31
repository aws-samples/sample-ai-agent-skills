---
title: "F1 — Deployment Package Size Issues"
description: "Diagnose Lambda deployment failures due to package size limits"
status: active
severity: MEDIUM
triggers:
  - "RequestEntityTooLargeException"
  - "Unzipped size must be smaller"
  - "Function code combined.*exceeds"
owner: devops-agent
objective: "Reduce package size to within Lambda limits"
context: "Limits: 50 MB zipped (direct upload), 250 MB unzipped (function + layers), 10 GB (container image). S3 upload bypasses the 50 MB direct upload limit but not the 250 MB unzipped limit."
---

## Common Issues

- symptoms: "RequestEntityTooLargeException on CreateFunction/UpdateFunctionCode"
  diagnosis: "Package exceeds 50 MB zipped for direct upload."
  resolution: "Upload to S3 first, then reference S3 location. Or use container image (up to 10 GB)."

- symptoms: "Unzipped size exceeds 250 MB"
  diagnosis: "Function code + all layers exceed 250 MB unzipped."
  resolution: "Remove unused dependencies, use tree-shaking/bundling, split into layers, or use container image."


## Safety Ratings

```
safety_ratings:
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
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
2. If layers were changed, revert to previous layer versions: `aws lambda update-function-configuration --function-name <name> --layers <previous-layer-arns>`
3. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<direct_upload_limit|unzipped_limit|layer_total>"
severity: MEDIUM
mitigation:
  immediate: "Upload via S3 or reduce package size"
  long_term: "Use bundlers (webpack/esbuild), container images for large packages"
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

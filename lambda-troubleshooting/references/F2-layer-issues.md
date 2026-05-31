---
title: "F2 — Lambda Layer Issues"
description: "Diagnose Lambda layer compatibility and loading issues"
status: active
severity: MEDIUM
triggers:
  - "cannot find module"
  - "layer.*not found"
  - "ImportError"
owner: devops-agent
objective: "Fix layer configuration and compatibility"
context: "Layers are extracted to /opt. Max 5 layers per function. Total unzipped (function + layers) ≤ 250 MB. Layer content must be in the correct directory structure: python/lib/python3.x/site-packages/ (Python), nodejs/node_modules/ (Node.js)."
---

## Common Issues

- symptoms: "Cannot find module, but it's in a layer"
  diagnosis: "Layer directory structure is wrong. Must follow runtime-specific paths."
  resolution: "Python: python/ or python/lib/python3.x/site-packages/. Node.js: nodejs/node_modules/."

- symptoms: "Layer version deleted or not accessible"
  diagnosis: "Layer version was deleted or function doesn't have permission to access cross-account layer."
  resolution: "Update function to use an existing layer version. Check layer permissions for cross-account."

- symptoms: "Function + layers exceed 250 MB"
  diagnosis: "Total unzipped size limit exceeded."
  resolution: "Consolidate layers, remove unused dependencies, or use container image."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
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

1. If function configuration was changed, revert using: `aws lambda update-function-configuration --function-name <name> --memory-size <original> --timeout <original>`
2. If a new version was published, point the alias back to the previous version: `aws lambda update-alias --function-name <name> --name <alias> --function-version <previous>`
3. If layers were changed, revert to previous layer versions: `aws lambda update-function-configuration --function-name <name> --layers <previous-layer-arns>`
4. If execution role was modified, revert IAM policy changes and verify function can still access required resources
5. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<wrong_directory_structure|layer_deleted|size_limit|permission>"
severity: MEDIUM
mitigation:
  immediate: "Fix layer structure or update layer version"
  long_term: "Automate layer builds in CI/CD with correct directory structure"
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

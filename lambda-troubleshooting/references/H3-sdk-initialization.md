---
title: "H3 — SDK / Client Initialization Overhead"
description: "Diagnose slow Lambda init caused by SDK client creation"
status: active
severity: LOW
triggers:
  - "Init Duration.*high"
  - "import.*slow"
owner: devops-agent
objective: "Optimize initialization for faster cold starts"
context: "SDK clients created in global scope persist across warm invocations. Creating them inside the handler wastes time on every invocation. Heavy imports (pandas, numpy, boto3) add to init time."
---

## Common Issues

- symptoms: "Init Duration high, many SDK clients created"
  diagnosis: "Multiple SDK clients initialized during cold start."
  resolution: "Lazy-initialize clients (create on first use). Only import what's needed."

- symptoms: "Every invocation is slow (not just cold starts)"
  diagnosis: "SDK clients created inside handler, not reused."
  resolution: "Move client creation to global scope outside the handler function."


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

1. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
2. Document all changes made during troubleshooting for audit trail
3. Verify service health after any rollback using monitoring dashboards
## Output Format

```yaml
root_cause: "<sdk_in_handler|heavy_imports|too_many_clients>"
severity: LOW
mitigation:
  immediate: "Move SDK clients to global scope"
  long_term: "Use lazy initialization, minimize imports, consider lighter SDKs"
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

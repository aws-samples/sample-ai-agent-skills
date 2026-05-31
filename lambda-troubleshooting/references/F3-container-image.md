---
title: "F3 — Container Image Deployment Issues"
description: "Diagnose Lambda container image deployment and runtime failures"
status: active
severity: MEDIUM
triggers:
  - "ImageNotFound"
  - "container image.*error"
  - "Runtime Interface Client"
owner: devops-agent
objective: "Fix container image deployment for Lambda"
context: "Lambda container images must implement the Lambda Runtime Interface Client (RIC). Max 10 GB. Must be in ECR (same account/region or cross-account with permissions). Base images available from AWS for each runtime."
---

## Common Issues

- symptoms: "Image not found or access denied"
  diagnosis: "Image not in ECR, wrong URI, or Lambda doesn't have ECR pull permissions."
  resolution: "Verify image URI. Ensure execution role has ecr:GetDownloadUrlForLayer, ecr:BatchGetImage."

- symptoms: "Function fails immediately with Runtime Interface Client error"
  diagnosis: "Container image doesn't implement Lambda RIC. Must use AWS base image or install RIC."
  resolution: "Use AWS base image (public.ecr.aws/lambda/) or install Lambda RIC in custom image."


## Safety Ratings

```
safety_ratings:
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
2. If layers were changed, revert to previous layer versions: `aws lambda update-function-configuration --function-name <name> --layers <previous-layer-arns>`
3. If execution role was modified, revert IAM policy changes and verify function can still access required resources
4. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<image_not_found|ecr_permission|missing_ric|wrong_entrypoint>"
severity: MEDIUM
mitigation:
  immediate: "Fix image URI, permissions, or add Lambda RIC"
  long_term: "Use AWS base images, automate ECR push in CI/CD"
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

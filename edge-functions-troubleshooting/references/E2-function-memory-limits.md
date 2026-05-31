---
title: "E2 — Function Memory Limits"
description: "Diagnose edge function memory limit issues"
status: active
severity: HIGH
triggers: ["out of memory", "memory limit exceeded", "package too large", "function size error"]
owner: devops-agent
objective: "Resolve edge function memory and size limit issues"
context: "CF Functions: 2MB package, 10KB code. Lambda@Edge: 128MB memory (viewer), 10GB memory (origin), 1MB package (viewer), 50MB package (origin)."
---

## Phase 1 — Triage

MUST:
- Check function package size: For Lambda@Edge: `aws lambda get-function --function-name <function-name> --region us-east-1 --query 'Configuration.CodeSize'`
- For CF Functions, check code size: `aws cloudfront describe-function --name <function-name> --stage LIVE`
- Check Lambda@Edge memory configuration: `aws lambda get-function-configuration --function-name <function-name> --region us-east-1 --query 'MemorySize'`

SHOULD:
- Check Lambda@Edge memory utilization in logs (Max Memory Used)
- Identify large dependencies in the deployment package
- Verify the function is using the correct event type for its size requirements

MAY:
- Analyze the deployment package contents for optimization opportunities
- Profile memory usage with representative inputs

## Phase 2 — Remediate

MUST:
- Reduce package size: tree-shake dependencies, remove unused code
- For Lambda@Edge viewer events: keep package under 1MB compressed
- For Lambda@Edge origin events: keep package under 50MB compressed

SHOULD:
- Increase Lambda@Edge memory if under-provisioned (up to 128MB for viewer, 10GB for origin)
- Use bundlers (webpack, esbuild) to minimize deployment package
- Remove development dependencies from production packages

MAY:
- Split functionality across multiple functions
- Use native Node.js/Python modules instead of large third-party libraries

## Common Issues

- symptoms: "Lambda@Edge deployment fails — package exceeds 1MB for viewer event"
  diagnosis: "Deployment package too large for viewer-request/viewer-response events."
  resolution: "Reduce package size with tree-shaking. Move to origin event if more space needed (50MB limit)."

- symptoms: "Lambda@Edge function crashes with 'Runtime exited with error: signal: killed'"
  diagnosis: "Function exceeded memory limit during execution."
  resolution: "Increase memory allocation. Optimize data processing to reduce memory usage."

## Safety Ratings

```
safety_ratings:
  - "lambda get-function: GREEN — read-only"
  - "lambda get-function-configuration: GREEN — read-only"
  - "cloudfront describe-function: GREEN — read-only"
  - "lambda update-function-configuration: YELLOW — changes memory allocation"
```

## Escalation Conditions

- Package cannot be reduced below the size limit for the required event type
- Memory requirements exceed the maximum for the event type (128MB viewer, 10GB origin)
- Function requires large dependencies that cannot be tree-shaken
- Memory optimization requires significant code refactoring

## Data Sensitivity

- `aws lambda get-function`: MEDIUM — reveals function configuration
- `aws lambda get-function-configuration`: MEDIUM — reveals memory and runtime settings
- `aws cloudfront describe-function`: LOW — function metadata
- Deployment package analysis: MEDIUM — reveals dependencies and code structure

## Prohibited Actions

- NEVER set Lambda@Edge memory below 128MB — it's the minimum and causes poor performance
- NEVER include node_modules or venv in deployment packages without pruning
- NEVER embed large data files in the deployment package — use S3 or DynamoDB
- NEVER ignore memory warnings in logs — they indicate imminent OOM crashes

## Phase 3 — Rollback

1. If memory was changed, restore previous setting: `aws lambda update-function-configuration --function-name <name> --memory-size <previous-mb> --region us-east-1`
2. If package was updated, deploy previous version
3. Verify function operates correctly after rollback
4. Document the memory/size analysis and optimization plan
5. Plan migration to origin event type if viewer limits are insufficient

## Output Format

```yaml
root_cause: "memory_limit — <specific_cause>"
evidence:
  - type: memory_error
    content: "<memory usage and package size details>"
severity: HIGH
mitigation:
  immediate: "Optimize package size or increase memory"
  long_term: "Implement build pipeline with size validation"
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
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"

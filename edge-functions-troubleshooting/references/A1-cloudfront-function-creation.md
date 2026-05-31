---
title: "A1 — CloudFront Function Creation"
description: "Diagnose CloudFront Function creation failures"
status: active
severity: HIGH
triggers: ["cannot create function", "function creation failed", "CreateFunction error"]
owner: devops-agent
objective: "Resolve CloudFront Function creation issues"
context: "CloudFront Functions are lightweight JavaScript functions running at edge locations. They use ECMAScript 5.1 and have strict size/time limits."
---

## Phase 1 — Triage

MUST:
- List existing functions: `aws cloudfront list-functions`
- Check IAM permissions for cloudfront:CreateFunction
- Verify function code is valid ECMAScript 5.1 JavaScript

SHOULD:
- Check function code size (max 10KB)
- Verify the function has a handler function named `handler`
- Check for syntax errors in the function code

MAY:
- Review CloudTrail for CreateFunction errors
- Check service quotas for CloudFront Functions per account

## Phase 2 — Remediate

MUST:
- Fix JavaScript syntax errors in the function code
- Ensure function exports a `handler(event)` function
- Reduce code size below 10KB if exceeding limit

SHOULD:
- Test function locally before creating
- Use the DEVELOPMENT stage for initial testing

MAY:
- Use CloudFormation or CDK for repeatable function deployment
- Implement a CI/CD pipeline for function updates

## Common Issues

- symptoms: "CreateFunction returns InvalidArgument — function code is invalid"
  diagnosis: "JavaScript syntax error or use of unsupported ES6+ features."
  resolution: "Rewrite using ECMAScript 5.1 syntax. Use var instead of let/const, function declarations instead of arrow functions."

- symptoms: "CreateFunction returns EntitySizeTooLarge"
  diagnosis: "Function code exceeds the 10KB size limit."
  resolution: "Minify the code. Remove comments and unnecessary whitespace. Simplify logic."

## Safety Ratings

```
safety_ratings:
  - "cloudfront list-functions: GREEN — read-only enumeration"
  - "cloudfront create-function: YELLOW — creates a new function in DEVELOPMENT stage"
  - "cloudfront test-function: GREEN — tests without affecting production"
  - "cloudfront publish-function: RED — makes function available for LIVE association"
```

## Escalation Conditions

- Function creation fails with internal service errors after fixing code issues
- Service quota limit reached for CloudFront Functions
- Function logic requires features not available in ECMAScript 5.1
- Function code size cannot be reduced below 10KB

## Data Sensitivity

- `aws cloudfront list-functions`: LOW — function metadata only
- `aws cloudfront create-function`: MEDIUM — function code may contain business logic
- `aws cloudfront test-function`: MEDIUM — test events may contain request data
- Function source code: MEDIUM — may contain routing logic or security rules

## Prohibited Actions

- NEVER publish a function to LIVE without testing in DEVELOPMENT stage
- NEVER include secrets or API keys directly in function code
- NEVER create functions with overly broad request/response manipulation
- NEVER bypass the DEVELOPMENT stage testing workflow

## Phase 3 — Rollback

1. If a function was created in error, delete it: `aws cloudfront delete-function --name <name> --if-match <etag>` (only if not associated with any distribution)
2. If function code needs to be reverted, update with previous code: `aws cloudfront update-function --name <name> --function-code fileb://previous-code.js --function-config Comment="rollback",Runtime=cloudfront-js-2.0 --if-match <etag>`
3. Verify no distributions reference the deleted/modified function
4. Document the creation issue and resolution
5. Update the function's test suite to prevent recurrence

## Output Format

```yaml
root_cause: "function_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix function code"
  long_term: "Implement CI/CD with syntax validation"
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

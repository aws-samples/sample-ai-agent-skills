---
title: "B2 — Transformation Failures"
description: "Diagnose data transformation failures in Object Lambda functions"
status: active
severity: HIGH
triggers: ["transformation failed", "incorrect output", "corrupted data", "wrong content returned"]
owner: devops-agent
objective: "Resolve data transformation issues in Object Lambda functions"
context: "Lambda functions transform S3 object data before returning it. Transformation bugs cause incorrect, corrupted, or missing data."
---

## Phase 1 — Triage

MUST:
- Review Lambda function logs for transformation errors: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms> --filter-pattern "ERROR"`
- Check Lambda duration metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Duration --dimensions Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
- Verify the function fetches the original object using inputS3Url from the event

SHOULD:
- Test the transformation logic with a known input object
- Check if the function handles different content types correctly
- Verify WriteGetObjectResponse includes correct Content-Type and Content-Length headers

MAY:
- Enable X-Ray tracing to identify slow transformation steps
- Compare transformed output against expected output for a test object

## Phase 2 — Remediate

MUST:
- Fix transformation logic bugs identified in logs
- Ensure the function uses inputS3Url to fetch the original object (not direct S3 API calls)
- Set correct response headers in WriteGetObjectResponse (Content-Type, Content-Length, statusCode)

SHOULD:
- Add input validation for the object data before transformation
- Implement error handling that returns meaningful error responses via WriteGetObjectResponse
- Add logging for transformation input/output sizes

MAY:
- Implement streaming transformation for large objects
- Add unit tests for the transformation logic

## Common Issues

- symptoms: "Transformed data is truncated or corrupted"
  diagnosis: "Content-Length header does not match actual body size, or encoding issues."
  resolution: "Calculate Content-Length from the transformed body. Ensure binary data is handled correctly (Buffer, not string)."

- symptoms: "Transformation works for small objects but fails for large ones"
  diagnosis: "Lambda memory insufficient or 60-second Object Lambda timeout exceeded."
  resolution: "Increase Lambda memory. Optimize transformation for streaming. Ensure completion within 60 seconds."

## Safety Ratings

```
safety_ratings:
  - "logs filter-log-events: GREEN — read-only log search"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "lambda invoke with test payload: YELLOW — executes transformation code"
  - "lambda update-function-code: RED — changes production function behavior"
```

## Escalation Conditions

- Transformation produces incorrect results that cannot be debugged from logs
- Data corruption affects downstream systems or end users
- Transformation requires more than 60 seconds (Object Lambda limit)
- Function requires access to external services that are unavailable

## Data Sensitivity

- `aws logs filter-log-events`: HIGH — logs may contain transformed object data or PII
- `aws lambda invoke`: HIGH — test payloads may trigger processing of sensitive data
- `aws cloudwatch get-metric-statistics`: LOW — aggregated metrics only
- Transformed object content: HIGH — may contain sensitive business data

## Prohibited Actions

- NEVER log full object content in production Lambda functions
- NEVER bypass transformation by returning the original object without disclosure
- NEVER deploy untested transformation code to production
- NEVER ignore Content-Type mismatches — they cause downstream parsing failures

## Phase 3 — Rollback

1. Roll back Lambda function to previous working version: `aws lambda update-function-code --function-name <name> --s3-bucket <bucket> --s3-key <previous-key>`
2. If using aliases, point alias to previous version: `aws lambda update-alias --function-name <name> --name prod --function-version <previous-version>`
3. Verify transformed output is correct after rollback
4. Check for any data corruption caused by the faulty transformation
5. Document the transformation bug and add regression tests

## Output Format

```yaml
root_cause: "transformation_failure — <specific_cause>"
evidence:
  - type: transformation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix transformation logic or rollback"
  long_term: "Add transformation validation and testing pipeline"
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
  - command: "get-bucket-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-bucket-acl"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-public-access-block"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling S3 Block Public Access"
  - "NEVER suggest Principal: * without Condition keys"
  - "NEVER suggest removing bucket encryption"

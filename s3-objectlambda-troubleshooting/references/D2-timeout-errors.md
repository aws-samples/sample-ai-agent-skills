---
title: "D2 — Timeout Errors"
description: "Diagnose timeout errors in Object Lambda execution"
status: active
severity: HIGH
triggers: ["timeout", "504 error", "function timed out", "Object Lambda timeout"]
owner: devops-agent
objective: "Resolve timeout errors in S3 Object Lambda"
context: "Object Lambda enforces a 60-second timeout for WriteGetObjectResponse. This is independent of the Lambda function's own timeout setting."
---

## Phase 1 — Triage

MUST:
- Check Lambda function timeout setting: `aws lambda get-function-configuration --function-name <function-name> --query 'Timeout'`
- Check Lambda duration metrics for p99 and max: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Duration --dimensions Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics p99,Maximum`
- Review Lambda logs for timeout patterns: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms> --filter-pattern "Task timed out"`

SHOULD:
- Identify which phase is slow: object download, transformation, or WriteGetObjectResponse
- Check object sizes that trigger timeouts
- Verify network connectivity if Lambda is in a VPC

MAY:
- Enable X-Ray tracing to pinpoint the slow segment
- Test with smaller objects to isolate the bottleneck

## Phase 2 — Remediate

MUST:
- Optimize transformation to complete within 60 seconds (Object Lambda limit)
- Increase Lambda memory to get more CPU (memory and CPU scale together)
- Ensure Lambda function timeout is at least 60 seconds to avoid premature function termination

SHOULD:
- Implement streaming transformation instead of buffering entire objects
- Add timeout handling in the function to return a partial response or error before the 60-second limit
- Pre-process large objects asynchronously if real-time transformation is too slow

MAY:
- Split large transformations into multiple smaller operations
- Cache intermediate results in DynamoDB or ElastiCache

## Common Issues

- symptoms: "Lambda function times out at 3 seconds (default timeout)"
  diagnosis: "Lambda function timeout is set to default 3 seconds, which is too short for Object Lambda."
  resolution: "Increase Lambda timeout to at least 60 seconds: aws lambda update-function-configuration --function-name <name> --timeout 60."

- symptoms: "Object Lambda returns 504 but Lambda function completes successfully at 65 seconds"
  diagnosis: "Object Lambda 60-second timeout exceeded. Lambda function timeout is longer but Object Lambda cuts off at 60 seconds."
  resolution: "Optimize transformation to complete within 60 seconds. The Lambda timeout setting does not extend the Object Lambda limit."

## Safety Ratings

```
safety_ratings:
  - "lambda get-function-configuration: GREEN — read-only"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "logs filter-log-events: GREEN — read-only log search"
  - "lambda update-function-configuration: YELLOW — changes function settings"
```

## Escalation Conditions

- Transformation cannot be optimized below 60 seconds for required object sizes
- Streaming transformation is not feasible for the transformation type
- VPC networking adds significant latency that cannot be reduced
- Business requirements demand real-time transformation of objects too large for the timeout

## Data Sensitivity

- `aws lambda get-function-configuration`: MEDIUM — reveals function settings
- `aws cloudwatch get-metric-statistics`: LOW — aggregated metrics
- `aws logs filter-log-events`: HIGH — may contain request data in timeout error context
- Performance profiling data: LOW — timing data only

## Prohibited Actions

- NEVER set Lambda function timeout below 60 seconds for Object Lambda functions
- NEVER assume increasing Lambda timeout beyond 60 seconds fixes Object Lambda timeouts
- NEVER return empty responses to avoid timeouts — return proper error responses
- NEVER disable timeout handling — always implement graceful timeout behavior

## Phase 3 — Rollback

1. If Lambda timeout was changed, restore previous value: `aws lambda update-function-configuration --function-name <name> --timeout <previous-seconds>`
2. If Lambda memory was increased, restore if causing cost concerns: `aws lambda update-function-configuration --function-name <name> --memory-size <previous-mb>`
3. If function code was modified for optimization, roll back if broken
4. Verify timeout errors are resolved or at previous frequency
5. Document the timeout analysis and optimization attempts

## Output Format

```yaml
root_cause: "timeout — <specific_cause>"
evidence:
  - type: timeout_error
    content: "<duration metrics and timeout details>"
severity: HIGH
mitigation:
  immediate: "Optimize function or increase resources"
  long_term: "Implement streaming transformation and pre-processing pipeline"
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

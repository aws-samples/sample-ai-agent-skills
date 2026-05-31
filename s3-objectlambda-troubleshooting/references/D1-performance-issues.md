---
title: "D1 — Performance Issues"
description: "Diagnose performance degradation in Object Lambda access"
status: active
severity: MEDIUM
triggers: ["slow response", "high latency", "performance degradation", "OLAP slow"]
owner: devops-agent
objective: "Resolve performance issues with S3 Object Lambda"
context: "Object Lambda adds latency from Lambda invocation, object retrieval, transformation, and WriteGetObjectResponse. Each step can be a bottleneck."
---

## Phase 1 — Triage

MUST:
- Check Lambda duration metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Duration --dimensions Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics Average,p99,Maximum`
- Check for cold starts: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name InitDuration --dimensions Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
- Check Object Lambda request latency: `aws cloudwatch get-metric-statistics --namespace AWS/S3ObjectLambda --metric-name FirstByteLatency --dimensions Name=AccessPointName,Value=<olap-name> --start-time <start> --end-time <end> --period 300 --statistics Average,p99`

SHOULD:
- Check Lambda memory utilization to identify under-provisioned functions
- Verify the function is not downloading unnecessarily large objects
- Check for Lambda throttling

MAY:
- Enable X-Ray tracing to identify specific bottlenecks
- Profile the transformation logic for optimization opportunities

## Phase 2 — Remediate

MUST:
- Increase Lambda memory if memory-bound (also increases CPU proportionally)
- Optimize transformation logic to reduce execution time
- Ensure the function stays within the 60-second Object Lambda timeout

SHOULD:
- Enable provisioned concurrency to eliminate cold starts: `aws lambda put-provisioned-concurrency-config --function-name <name> --qualifier <alias> --provisioned-concurrent-executions <count>`
- Use streaming for large object transformations
- Cache frequently accessed data in Lambda /tmp or external cache

MAY:
- Implement connection pooling for external service calls
- Use Lambda Layers for shared dependencies to reduce package size

## Common Issues

- symptoms: "First request after idle period is very slow (5-10 seconds)"
  diagnosis: "Lambda cold start — function initialization takes time."
  resolution: "Enable provisioned concurrency or use smaller deployment packages. Optimize initialization code."

- symptoms: "Latency increases linearly with object size"
  diagnosis: "Function downloads entire object before transforming."
  resolution: "Implement streaming transformation. Use range requests if only part of the object is needed."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "lambda put-provisioned-concurrency-config: YELLOW — adds cost, changes scaling behavior"
  - "lambda update-function-configuration: YELLOW — changes function settings"
  - "xray get-trace-summaries: GREEN — read-only tracing data"
```

## Escalation Conditions

- Latency exceeds 60 seconds (Object Lambda timeout) and cannot be optimized
- Cold start latency is unacceptable and provisioned concurrency cost is prohibitive
- Transformation requires external service calls with unpredictable latency
- Performance requirements cannot be met with current Object Lambda architecture

## Data Sensitivity

- `aws cloudwatch get-metric-statistics`: LOW — aggregated metrics only
- `aws xray get-trace-summaries`: MEDIUM — may reveal request patterns and service dependencies
- `aws lambda get-function-configuration`: MEDIUM — reveals memory, timeout, and environment settings
- Performance profiling data: LOW — timing data without content

## Prohibited Actions

- NEVER set Lambda memory below 128 MB — it causes severe performance degradation
- NEVER disable provisioned concurrency during peak hours without a maintenance window
- NEVER optimize by skipping transformation — this changes the functional behavior
- NEVER cache sensitive transformed data in unencrypted external stores

## Phase 3 — Rollback

1. If memory was changed, restore previous setting: `aws lambda update-function-configuration --function-name <name> --memory-size <previous-mb>`
2. If provisioned concurrency was added and causing issues, remove it: `aws lambda delete-provisioned-concurrency-config --function-name <name> --qualifier <alias>`
3. If function code was optimized and broke, roll back to previous version
4. Verify latency returns to previous baseline after rollback
5. Document performance findings for future optimization attempts

## Output Format

```yaml
root_cause: "performance — <specific_cause>"
evidence:
  - type: latency_metrics
    content: "<metric details>"
severity: MEDIUM
mitigation:
  immediate: "Optimize Lambda configuration"
  long_term: "Implement streaming, caching, and provisioned concurrency"
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

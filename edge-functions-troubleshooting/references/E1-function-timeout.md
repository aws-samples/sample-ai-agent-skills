---
title: "E1 — Function Timeout"
description: "Diagnose edge function timeout errors"
status: active
severity: HIGH
triggers: ["function timeout", "execution time exceeded", "1ms limit", "5s timeout", "30s timeout"]
owner: devops-agent
objective: "Resolve edge function timeout issues"
context: "CF Functions: 1ms limit. Lambda@Edge viewer: 5s. Lambda@Edge origin: 30s. These are hard limits that cannot be increased."
---

## Phase 1 — Triage

MUST:
- Identify function type and event type to determine the timeout limit
- For CF Functions, check compute utilization: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name FunctionComputeUtilization --dimensions Name=DistributionId,Value=<dist-id>,Name=FunctionName,Value=<func-name> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
- For Lambda@Edge, check duration: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Duration --dimensions Name=FunctionName,Value=us-east-1.<function-name> --start-time <start> --end-time <end> --period 300 --statistics Average,p99,Maximum --region <edge-region>`

SHOULD:
- Identify which operations consume the most time
- Check if timeouts correlate with specific request patterns or object sizes
- Verify network calls (Lambda@Edge only) are not the bottleneck

MAY:
- Enable X-Ray tracing for Lambda@Edge to identify slow segments
- Profile the function locally with representative inputs

## Phase 2 — Remediate

MUST:
- Optimize function logic to fit within the timeout limit
- For CF Functions: simplify to basic operations (header manipulation, URL rewrite, simple redirects)
- For Lambda@Edge: add timeout handling to return gracefully before the hard limit

SHOULD:
- Move complex logic from CF Functions to Lambda@Edge (5s/30s vs 1ms)
- Cache external service responses in Lambda@Edge to reduce network call latency
- Implement circuit breakers for external service calls

MAY:
- Pre-compute results and store in DynamoDB/S3 for edge function lookup
- Split complex logic across multiple event types (viewer-request + origin-request)

## Common Issues

- symptoms: "CloudFront Function compute utilization consistently above 80%"
  diagnosis: "Function logic is too complex for the 1ms limit."
  resolution: "Simplify logic or migrate to Lambda@Edge. Remove loops, complex string operations, or regex."

- symptoms: "Lambda@Edge viewer-request times out at 5 seconds"
  diagnosis: "Function makes network calls that exceed the 5s viewer timeout."
  resolution: "Move network-dependent logic to origin-request (30s timeout) or pre-compute results."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics: GREEN — read-only"
  - "cloudfront test-function: GREEN — no production impact"
  - "lambda update-function-configuration: YELLOW — changes function settings"
  - "cloudfront update-distribution: RED — changes function association"
```

## Escalation Conditions

- Function cannot be optimized below the timeout limit for the required use case
- Timeout causes complete request failure with no fallback
- Migration from CF Functions to Lambda@Edge requires significant code rewrite
- External service latency is unpredictable and causes intermittent timeouts

## Data Sensitivity

- `aws cloudwatch get-metric-statistics`: LOW — aggregated metrics
- Performance profiling data: LOW — timing data only
- `aws xray get-trace-summaries`: MEDIUM — reveals request flow and service dependencies
- Function execution logs: HIGH — may contain request data

## Prohibited Actions

- NEVER attempt to increase CF Function timeout — it is a fixed 1ms limit
- NEVER attempt to increase Lambda@Edge timeout beyond 5s (viewer) or 30s (origin)
- NEVER add sleep/delay to functions to work around timing issues
- NEVER ignore timeout errors — they indicate the function is not suitable for the compute model

## Phase 3 — Rollback

1. Revert function code to previous working version
2. If migrating between function types, restore the original association
3. Verify timeout errors are resolved after rollback
4. Document the timeout analysis and optimization attempts
5. Plan migration to appropriate compute model if current one is insufficient

## Output Format

```yaml
root_cause: "timeout — <specific_cause>"
evidence:
  - type: timeout_metrics
    content: "<compute utilization or duration metrics>"
severity: HIGH
mitigation:
  immediate: "Optimize function or migrate to appropriate compute model"
  long_term: "Pre-compute results and implement caching strategies"
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

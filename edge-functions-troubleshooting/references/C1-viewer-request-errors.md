---
title: "C1 — Viewer Request Errors"
description: "Diagnose viewer-request event function errors"
status: active
severity: HIGH
triggers: ["viewer request error", "request function failed", "502 from edge function", "viewer-request timeout"]
owner: devops-agent
objective: "Resolve viewer-request event function errors"
context: "Viewer-request functions run before CloudFront checks the cache. Errors here affect every request. CloudFront Functions have 1ms limit; Lambda@Edge has 5s limit."
---

## Phase 1 — Triage

MUST:
- Identify which function type is associated (CF Function or Lambda@Edge): `aws cloudfront get-distribution-config --id <distribution-id>`
- For CF Functions, check execution errors: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name FunctionExecutionErrors --dimensions Name=DistributionId,Value=<dist-id>,Name=FunctionName,Value=<func-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- For Lambda@Edge, check logs in the region closest to the user: `aws logs filter-log-events --log-group-name /aws/lambda/us-east-1.<function-name> --start-time <epoch-ms> --filter-pattern "ERROR" --region <edge-region>`

SHOULD:
- Check function compute utilization (CF Functions): `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name FunctionComputeUtilization --dimensions Name=DistributionId,Value=<dist-id>,Name=FunctionName,Value=<func-name> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
- Verify the function returns a valid request or response object
- Check for unhandled exceptions in the function code

MAY:
- Test CF Function with test event: `aws cloudfront test-function --name <function-name> --if-match <etag> --stage LIVE --event-object fileb://test-event.json`
- Reproduce the error with a specific request pattern

## Phase 2 — Remediate

MUST:
- Fix function code errors (syntax, runtime, logic)
- Ensure function returns a valid request object to continue to cache/origin
- For CF Functions: optimize to stay within 1ms compute time

SHOULD:
- Add error handling to return a graceful error response instead of 502
- Implement input validation for request headers and URI
- Test with representative request patterns before deploying

MAY:
- Implement request logging for debugging (be mindful of PII)
- Add CloudWatch alarms for function error rates

## Common Issues

- symptoms: "All requests return 502 Bad Gateway"
  diagnosis: "Viewer-request function throws an unhandled exception."
  resolution: "Wrap function logic in try-catch. Return a valid request object or error response."

- symptoms: "CloudFront Function exceeds compute utilization (>80%)"
  diagnosis: "Function logic is too complex for the 1ms time limit."
  resolution: "Simplify logic. Move complex operations to Lambda@Edge or origin."

## Safety Ratings

```
safety_ratings:
  - "cloudfront get-distribution-config: GREEN — read-only"
  - "cloudwatch get-metric-statistics: GREEN — read-only"
  - "cloudfront test-function: GREEN — no production impact"
  - "cloudfront update-distribution: RED — affects all viewer requests"
```

## Escalation Conditions

- Viewer-request errors cause complete site outage (all requests fail)
- Function error rate exceeds 5% and cannot be quickly resolved
- Error only occurs for specific geographic regions or request patterns
- Function requires capabilities not available in the current compute model

## Data Sensitivity

- `aws cloudfront get-distribution-config`: MEDIUM — reveals distribution settings
- `aws logs filter-log-events`: HIGH — may contain request headers, URIs, and client data
- `aws cloudfront test-function`: MEDIUM — test events may contain request data
- Viewer request data: HIGH — contains client IPs, headers, cookies

## Prohibited Actions

- NEVER deploy viewer-request functions without thorough testing — they affect ALL requests
- NEVER log full request headers in production (may contain auth tokens, cookies)
- NEVER return sensitive error details to viewers in error responses
- NEVER remove viewer-request functions during peak traffic without a maintenance window

## Phase 3 — Rollback

1. For CF Functions: publish previous working version or disassociate the function
2. For Lambda@Edge: update distribution to use previous function version
3. Wait for distribution to reach 'Deployed' status
4. Verify all requests succeed after rollback
5. Document the error pattern and add to test suite

## Output Format

```yaml
root_cause: "viewer_request — <specific_cause>"
evidence:
  - type: function_error
    content: "<error details from logs/metrics>"
severity: HIGH
mitigation:
  immediate: "Fix function or rollback"
  long_term: "Implement comprehensive testing and error handling"
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

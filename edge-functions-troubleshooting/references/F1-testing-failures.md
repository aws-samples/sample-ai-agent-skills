---
title: "F1 — Testing Failures"
description: "Diagnose edge function testing failures"
status: active
severity: MEDIUM
triggers: ["test failed", "test-function error", "function test not working", "test event invalid"]
owner: devops-agent
objective: "Resolve edge function testing issues"
context: "CF Functions can be tested with test-function API. Lambda@Edge testing requires local invocation or test distributions. Test events must match the expected event structure."
---

## Phase 1 — Triage

MUST:
- For CF Functions, run test: `aws cloudfront test-function --name <function-name> --if-match <etag> --stage DEVELOPMENT --event-object fileb://test-event.json`
- Verify test event structure matches the expected CloudFront event format
- Check test output for function errors vs test configuration errors

SHOULD:
- Verify the test event includes all required fields (request.uri, request.method, request.headers)
- Test with multiple event types (viewer-request, viewer-response)
- Check if the function handles edge cases (missing headers, empty values)

MAY:
- Create a test distribution for Lambda@Edge testing
- Use local testing frameworks (SAM local, jest) for Lambda@Edge

## Phase 2 — Remediate

MUST:
- Fix test event structure to match CloudFront event format
- Fix function code errors identified in test output
- Ensure test events cover the function's expected input patterns

SHOULD:
- Create a comprehensive test suite covering normal and error cases
- Test with realistic request patterns from production logs
- Validate function output matches expected response format

MAY:
- Implement automated testing in CI/CD pipeline
- Use CloudFront continuous deployment for production-like testing

## Common Issues

- symptoms: "test-function returns TestFailed with 'Unexpected token' error"
  diagnosis: "Function code has JavaScript syntax errors."
  resolution: "Fix syntax errors. For CF Functions, ensure ECMAScript 5.1 compatibility."

- symptoms: "test-function returns success but function fails in production"
  diagnosis: "Test event does not match real CloudFront event structure."
  resolution: "Use CloudFront real-time logs to capture actual event structure. Update test events accordingly."

## Safety Ratings

```
safety_ratings:
  - "cloudfront test-function: GREEN — no production impact"
  - "lambda invoke: YELLOW — executes function code"
  - "cloudfront describe-function: GREEN — read-only"
  - "logs filter-log-events: GREEN — read-only"
```

## Escalation Conditions

- Function passes all tests but fails in production
- Test environment cannot replicate production conditions
- Testing requires access to production data or services
- Automated testing pipeline is broken or unreliable

## Data Sensitivity

- `aws cloudfront test-function`: MEDIUM — test events may contain request data
- Test event files: MEDIUM — may contain realistic request patterns
- Test output: MEDIUM — may reveal function logic and error details
- Production log samples for test creation: HIGH — contain real user data

## Prohibited Actions

- NEVER use production user data in test events without anonymization
- NEVER skip testing before publishing to LIVE stage
- NEVER assume passing tests guarantee production success
- NEVER test against production distributions without a rollback plan

## Phase 3 — Rollback

1. If a function was published based on faulty tests, rollback the LIVE version
2. Fix the test suite to catch the production failure
3. Re-test with corrected test events
4. Verify the function passes updated tests before republishing
5. Document the testing gap and update the test strategy

## Output Format

```yaml
root_cause: "testing — <specific_cause>"
evidence:
  - type: test_failure
    content: "<test output details>"
severity: MEDIUM
mitigation:
  immediate: "Fix test events or function code"
  long_term: "Implement comprehensive automated testing pipeline"
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

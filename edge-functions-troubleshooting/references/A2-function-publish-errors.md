---
title: "A2 — Function Publish Errors"
description: "Diagnose CloudFront Function publish failures from DEVELOPMENT to LIVE"
status: active
severity: HIGH
triggers: ["publish failed", "cannot publish function", "PublishFunction error", "function not going live"]
owner: devops-agent
objective: "Resolve CloudFront Function publish failures"
context: "Functions must be published from DEVELOPMENT to LIVE stage. Publishing validates the function and makes it available for distribution association."
---

## Phase 1 — Triage

MUST:
- Check function status: `aws cloudfront describe-function --name <function-name> --stage DEVELOPMENT`
- Verify function passes test: `aws cloudfront test-function --name <function-name> --if-match <etag> --stage DEVELOPMENT --event-object fileb://test-event.json`
- Check IAM permissions for cloudfront:PublishFunction

SHOULD:
- Verify the function ETag matches (concurrent modification check)
- Check if the function has validation errors
- Review recent function updates that may have introduced issues

MAY:
- Compare DEVELOPMENT and LIVE versions to identify differences
- Check CloudTrail for PublishFunction error details

## Phase 2 — Remediate

MUST:
- Fix validation errors identified during publish
- Ensure function passes all test cases before publishing
- Use correct ETag in the publish request (get latest with describe-function)

SHOULD:
- Run comprehensive test cases covering edge cases
- Verify function behavior matches expected output

MAY:
- Implement automated testing before publish in CI/CD
- Use canary deployments by associating with a test distribution first

## Common Issues

- symptoms: "PublishFunction returns PreconditionFailed"
  diagnosis: "ETag mismatch — function was modified since last describe."
  resolution: "Get the latest ETag: aws cloudfront describe-function --name <name> --stage DEVELOPMENT. Use the returned ETag."

- symptoms: "PublishFunction returns InvalidFunction — validation failed"
  diagnosis: "Function code has runtime errors that pass syntax check but fail validation."
  resolution: "Test the function thoroughly: aws cloudfront test-function. Fix runtime errors in the handler."

## Safety Ratings

```
safety_ratings:
  - "cloudfront describe-function: GREEN — read-only"
  - "cloudfront test-function: GREEN — tests without affecting production"
  - "cloudfront publish-function: RED — makes function available for LIVE traffic"
  - "cloudfront update-function: YELLOW — modifies function code"
```

## Escalation Conditions

- Function passes all tests but publish still fails
- Publish succeeds but function behaves differently in LIVE vs DEVELOPMENT
- Publishing causes errors on associated distributions
- Rollback to previous LIVE version is needed urgently

## Data Sensitivity

- `aws cloudfront describe-function`: LOW — function metadata
- `aws cloudfront test-function`: MEDIUM — test events may contain request data
- `aws cloudfront publish-function`: MEDIUM — changes production behavior
- Function test results: LOW — synthetic test data

## Prohibited Actions

- NEVER publish a function without running test cases first
- NEVER force-publish by bypassing ETag checks
- NEVER publish during peak traffic without a rollback plan
- NEVER publish functions that modify security headers without security review

## Phase 3 — Rollback

1. If a bad function was published, update with previous code and republish: `aws cloudfront update-function --name <name> --function-code fileb://previous-code.js --function-config Comment="rollback",Runtime=cloudfront-js-2.0 --if-match <etag>` then `aws cloudfront publish-function --name <name> --if-match <new-etag>`
2. If rollback is urgent, disassociate the function from the cache behavior
3. Verify production traffic is unaffected after rollback
4. Document the publish failure and root cause
5. Add the failing scenario to the test suite

## Output Format

```yaml
root_cause: "publish_error — <specific_cause>"
evidence:
  - type: publish_failure
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix and republish or rollback"
  long_term: "Implement automated testing gate before publish"
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

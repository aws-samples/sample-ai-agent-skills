---
title: "F1 — API Integration"
description: "Diagnose WorkDocs API integration issues"
status: active
severity: MEDIUM
triggers: ["API error", "SDK error", "integration failed", "authentication token", "API throttling"]
owner: devops-agent
objective: "Resolve API integration issues including authentication, rate limiting, and SDK configuration"
context: "WorkDocs API uses IAM for admin operations and authentication tokens for user operations. Rate limits apply to all API calls."
---

## Phase 1 — Triage

MUST:
- Verify IAM permissions for the calling principal: `aws sts get-caller-identity`
- Check if the API call requires an authentication token (user-level operations)
- Verify the organization ID is correct: `aws workdocs describe-users --organization-id <org-id> --limit 1`
- Check for throttling errors in CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=workdocs.amazonaws.com --max-results 20`

SHOULD:
- Verify SDK version is current and supports the API operations being used
- Check API endpoint region matches the WorkDocs site region
- Review error response codes and messages for specific guidance

MAY:
- Enable SDK debug logging to capture request/response details
- Check AWS Service Health Dashboard for API availability issues

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for admin API operations
- Obtain valid authentication tokens for user-level operations
- Implement exponential backoff for throttled requests

SHOULD:
- Update SDK to the latest version
- Use the correct regional endpoint for API calls
- Cache authentication tokens to reduce token generation overhead

MAY:
- Implement request queuing to stay within rate limits
- Set up CloudWatch alarms for API error rates

## Safety Ratings

```
safety_ratings:
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "workdocs describe-users: GREEN — read-only user listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "IAM policy changes: RED — modifies access permissions"
```

## Escalation Conditions

- API returns 500 errors consistently suggesting service-side issues
- Rate limits are too low for the integration workload and cannot be increased
- Authentication token generation fails despite correct credentials
- API behavior differs from documentation

## Data Sensitivity

- `aws sts get-caller-identity`: LOW — returns caller identity only
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call patterns
- Authentication tokens: HIGH — provide user-level access to WorkDocs content
- SDK debug logs: MEDIUM — may contain request parameters and headers

## Prohibited Actions

- NEVER hardcode authentication tokens in application code
- NEVER log authentication tokens or IAM credentials
- NEVER bypass rate limits by using multiple IAM principals
- NEVER use admin API operations when user-level operations are appropriate

## Phase 3 — Rollback

1. If IAM permissions were changed, revert to original policy
2. If SDK version was updated, revert if the new version introduces regressions
3. If API integration code was modified, restore from version control
4. Invalidate any authentication tokens that were exposed during debugging
5. Document API integration configuration and authentication flow

## Common Issues

- symptoms: "API returns 'UnauthorizedResourceAccessException'"
  diagnosis: "Missing authentication token for user-level operation or insufficient IAM permissions."
  resolution: "For user operations, include authentication token header. For admin operations, verify IAM permissions."

- symptoms: "API returns 'TooManyRequestsException'"
  diagnosis: "API rate limit exceeded."
  resolution: "Implement exponential backoff with jitter. Reduce request frequency or batch operations."

## Output Format

```yaml
root_cause: "api_integration — <specific_cause>"
evidence:
  - type: api_error
    content: "<error code, message, and request details>"
severity: MEDIUM
mitigation:
  immediate: "Fix API authentication or rate limiting"
  long_term: "Implement robust error handling and monitoring"
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

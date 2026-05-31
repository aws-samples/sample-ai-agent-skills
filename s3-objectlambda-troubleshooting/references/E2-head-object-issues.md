---
title: "E2 — HEAD Object Issues"
description: "Diagnose HEAD object request failures through Object Lambda"
status: active
severity: MEDIUM
triggers: ["HEAD request failed", "metadata not returned", "HeadObject error", "HEAD returns wrong headers"]
owner: devops-agent
objective: "Resolve HEAD object request issues with Object Lambda"
context: "HEAD requests through OLAPs invoke the Lambda function with a HeadObject event type. The function must respond with headers only, no body."
---

## Phase 1 — Triage

MUST:
- Check OLAP configuration includes HeadObject action: `aws s3control get-access-point-configuration-for-object-lambda --account-id <account-id> --name <olap-name>`
- Test HEAD request: `aws s3api head-object --bucket <olap-arn> --key <key>`
- Review Lambda logs for HeadObject event handling: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --filter-pattern "HeadObject"`

SHOULD:
- Verify the Lambda function detects the event type (GetObject vs HeadObject)
- Check if the function returns appropriate headers (Content-Type, Content-Length, ETag)
- Verify the function does NOT return a body for HEAD requests

MAY:
- Compare HEAD response headers with direct S3 HEAD response
- Test with different object types to identify pattern-specific failures

## Phase 2 — Remediate

MUST:
- Add HeadObject to OLAP TransformationConfigurations actions if missing
- Update Lambda function to detect HeadObject event type and respond with headers only
- Ensure WriteGetObjectResponse is called with appropriate metadata headers

SHOULD:
- Return transformed metadata that matches what a GET would return (consistent Content-Length, Content-Type)
- Handle HeadObject-Range and HeadObject-PartNumber if AllowedFeatures includes them

MAY:
- Cache HEAD response metadata to reduce Lambda invocations
- Implement conditional HEAD responses (If-Modified-Since, If-None-Match)

## Common Issues

- symptoms: "HEAD request bypasses Lambda — returns original object metadata"
  diagnosis: "HeadObject is not configured as a transformation action in the OLAP."
  resolution: "Add HeadObject to the TransformationConfigurations actions list in the OLAP configuration."

- symptoms: "HEAD request returns 500 error"
  diagnosis: "Lambda function treats HeadObject like GetObject and tries to return a body."
  resolution: "Detect event type in the function. For HeadObject, call WriteGetObjectResponse with headers only (no body)."

## Safety Ratings

```
safety_ratings:
  - "get-access-point-configuration-for-object-lambda: GREEN — read-only"
  - "s3api head-object: GREEN — read-only metadata retrieval"
  - "logs filter-log-events: GREEN — read-only log search"
  - "put-access-point-configuration-for-object-lambda: YELLOW — modifies OLAP behavior"
```

## Escalation Conditions

- HEAD responses must return transformed metadata that requires running the full transformation
- Clients depend on specific HEAD response headers that the Lambda function cannot provide
- HEAD request latency is unacceptable for the use case
- HeadObject handling conflicts with GetObject handling in the same function

## Data Sensitivity

- `aws s3api head-object`: LOW — returns metadata only, no object content
- `aws s3control get-access-point-configuration-for-object-lambda`: LOW — configuration metadata
- `aws logs filter-log-events`: MEDIUM — may contain request metadata
- HEAD response headers: LOW — metadata only

## Prohibited Actions

- NEVER return object body content in HEAD responses — this violates HTTP semantics
- NEVER skip HeadObject handling — many clients use HEAD before GET for pre-flight checks
- NEVER return inconsistent metadata between HEAD and GET responses
- NEVER ignore Content-Length in HEAD responses — clients use it for download planning

## Phase 3 — Rollback

1. If OLAP configuration was modified, restore previous configuration
2. If Lambda function was updated, roll back to previous version
3. Verify HEAD requests return expected metadata after rollback
4. Check if any clients cached incorrect HEAD response metadata
5. Document the HeadObject handling requirements

## Output Format

```yaml
root_cause: "head_object — <specific_cause>"
evidence:
  - type: head_error
    content: "<HEAD request and response details>"
severity: MEDIUM
mitigation:
  immediate: "Fix HeadObject handling in Lambda function"
  long_term: "Implement consistent HEAD/GET response handling"
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

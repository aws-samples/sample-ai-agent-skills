---
title: "E1 — Range GET Issues"
description: "Diagnose range GET and part-number request failures through Object Lambda"
status: active
severity: MEDIUM
triggers: ["range request failed", "partial content error", "byte range not working", "part number error"]
owner: devops-agent
objective: "Resolve range GET and part-number request issues with Object Lambda"
context: "Range GET and part-number requests are forwarded to the Lambda function. The function must handle these explicitly or return the full object."
---

## Phase 1 — Triage

MUST:
- Check OLAP configuration for AllowedFeatures: `aws s3control get-access-point-configuration-for-object-lambda --account-id <account-id> --name <olap-name>`
- Verify GetObject-Range and GetObject-PartNumber are in AllowedFeatures
- Review Lambda function code for range/partNumber handling in the event payload

SHOULD:
- Test a range request through the OLAP: `aws s3api get-object --bucket <olap-arn> --key <key> --range bytes=0-1023 output.bin`
- Check Lambda logs for range-related errors
- Verify the function reads event.getObjectContext.inputS3Url with range parameters

MAY:
- Compare behavior with and without range headers
- Test with different range formats (bytes=0-1023, bytes=1024-, bytes=-512)

## Phase 2 — Remediate

MUST:
- Enable GetObject-Range in OLAP AllowedFeatures if missing
- Update Lambda function to detect and handle range headers from the event
- Ensure WriteGetObjectResponse returns correct Content-Range header for range requests

SHOULD:
- Implement proper 206 Partial Content responses for range requests
- Handle edge cases: invalid ranges, ranges beyond object size
- Return 416 Range Not Satisfiable for invalid ranges

MAY:
- Optimize range handling to only download the requested range from S3
- Cache range metadata for frequently accessed objects

## Common Issues

- symptoms: "Range GET returns full object instead of requested range"
  diagnosis: "GetObject-Range not in AllowedFeatures, or Lambda function ignores range headers."
  resolution: "Add GetObject-Range to AllowedFeatures. Update Lambda to read range from event and return partial content."

- symptoms: "Range GET returns 500 error"
  diagnosis: "Lambda function crashes when processing range headers."
  resolution: "Add error handling for range parsing. Ensure Content-Range header format is correct: bytes start-end/total."

## Safety Ratings

```
safety_ratings:
  - "get-access-point-configuration-for-object-lambda: GREEN — read-only"
  - "s3api get-object with range: GREEN — read-only data retrieval"
  - "put-access-point-configuration-for-object-lambda: YELLOW — modifies OLAP behavior"
  - "lambda update-function-code: RED — changes function behavior"
```

## Escalation Conditions

- Range handling requires complex transformation logic that cannot maintain byte-level accuracy
- Multi-part download clients depend on exact range semantics that transformation breaks
- Range requests are critical for video streaming and cannot tolerate any errors
- Performance of range-aware transformation is worse than full-object transformation

## Data Sensitivity

- `aws s3control get-access-point-configuration-for-object-lambda`: LOW — configuration metadata
- `aws s3api get-object`: HIGH — retrieves actual object content
- `aws logs filter-log-events`: HIGH — may contain range request details and object metadata
- Range request patterns: LOW — byte offset data only

## Prohibited Actions

- NEVER return full objects when a range was requested without informing the client (omitting Content-Range)
- NEVER ignore part-number requests — they are used by multipart download clients
- NEVER cache range responses without proper cache key including the range
- NEVER return 200 OK for range requests — must return 206 Partial Content

## Phase 3 — Rollback

1. If AllowedFeatures was modified, restore previous configuration
2. If Lambda function was updated for range handling, roll back to previous version
3. Verify range requests work as expected (or fail gracefully) after rollback
4. Check downstream clients for any data corruption from incorrect range responses
5. Document the range handling requirements for future implementation

## Output Format

```yaml
root_cause: "range_get — <specific_cause>"
evidence:
  - type: range_error
    content: "<range request and response details>"
severity: MEDIUM
mitigation:
  immediate: "Fix range handling in Lambda function"
  long_term: "Implement comprehensive range/part-number support with tests"
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

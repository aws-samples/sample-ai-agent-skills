---
title: "B1 — Put Object Failures"
description: "Diagnose MediaStore object upload (put) failures"
status: active
severity: HIGH
triggers: ["put object failed", "upload failed", "cannot write to mediastore", "put-object error"]
owner: devops-agent
objective: "Resolve MediaStore object upload failures"
context: "Object uploads use the mediastore-data API with the container-specific endpoint. Paths must start with /."
---

## Phase 1 — Triage

MUST:
- Verify container endpoint: `aws mediastore describe-container --container-name <name> --query 'Container.Endpoint'`
- Test put operation: `aws mediastore-data put-object --endpoint <endpoint> --body <file> --path /<path> --content-type <type>`
- Check IAM permissions for mediastore-data:PutObject

SHOULD:
- Verify container status is ACTIVE: `aws mediastore describe-container --container-name <name> --query 'Container.Status'`
- Check container policy allows PutObject for the caller

MAY:
- Check for throttling: review CloudWatch RequestCount and ThrottleCount metrics
- Verify the object path format (must start with /)

## Phase 2 — Remediate

MUST:
- Ensure the path starts with `/`: `--path /folder/video.mp4`
- Set the correct content-type header for the media type
- Verify both IAM policy and container policy allow the operation

SHOULD:
- Use chunked transfer encoding for large files
- Set cache-control headers for CDN integration

MAY:
- Implement retry logic with exponential backoff for transient failures

## Safety Ratings

```
safety_ratings:
  - "mediastore describe-container: GREEN — read-only container query"
  - "mediastore-data put-object: YELLOW — writes data to container"
  - "mediastore-data describe-object: GREEN — read-only object metadata"
  - "cloudwatch get-metric-statistics: GREEN — read-only metric query"
```

## Escalation Conditions

- Put operations fail consistently with 500 errors after verifying permissions
- Throttling prevents live video ingestion (time-sensitive)
- Large object uploads fail mid-transfer repeatedly
- Container endpoint is unreachable

## Data Sensitivity

- Container endpoint: LOW — required for data plane access
- Object content: HIGH — may contain live video content
- Object paths: LOW — file path structure
- Content-type headers: LOW — MIME type information

## Prohibited Actions

- NEVER upload test data to production containers without coordination
- NEVER overwrite live video segments without understanding the streaming workflow
- NEVER disable container policies to troubleshoot put failures
- NEVER share container endpoints with data plane credentials publicly

## Phase 3 — Rollback

1. Delete incorrectly uploaded objects: `aws mediastore-data delete-object --endpoint <endpoint> --path <path>`
2. If container policy was modified, restore previous policy
3. If IAM permissions were broadened, revert to least-privilege
4. Verify live video ingestion is not disrupted
5. Document the upload failure cause and resolution

## Common Issues

- symptoms: "PutObject returns 403 Forbidden"
  diagnosis: "IAM policy or container policy does not allow mediastore-data:PutObject for the caller."
  resolution: "Add mediastore-data:PutObject to IAM policy AND ensure container policy allows the principal."

- symptoms: "PutObject returns 400 Bad Request with path error"
  diagnosis: "Object path does not start with / or contains invalid characters."
  resolution: "Ensure path starts with /. Use URL-safe characters in path names."

## Output Format

```yaml
root_cause: "put_object — <specific_cause>"
evidence:
  - type: upload_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix permissions or path format"
  long_term: "Implement upload validation in application"
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

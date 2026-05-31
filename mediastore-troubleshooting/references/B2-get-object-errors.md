---
title: "B2 — Get Object Errors"
description: "Diagnose MediaStore object retrieval (get) failures"
status: active
severity: HIGH
triggers: ["get object failed", "download failed", "cannot read from mediastore", "object not found", "404 error"]
owner: devops-agent
objective: "Resolve MediaStore object retrieval failures"
context: "Object retrieval uses the mediastore-data API. Objects are accessed by path. MediaStore is optimized for low-latency reads."
---

## Phase 1 — Triage

MUST:
- Verify the object exists: `aws mediastore-data describe-object --endpoint <endpoint> --path /<path>`
- List objects in the path: `aws mediastore-data list-items --endpoint <endpoint> --path /`
- Check IAM permissions for mediastore-data:GetObject

SHOULD:
- Verify container policy allows GetObject for the caller
- Check if the object was recently deleted (no versioning — deletion is permanent)

MAY:
- Check access logs for the specific request: review CloudWatch Logs `/aws/mediastore/<container>`

## Phase 2 — Remediate

MUST:
- Verify the exact object path (case-sensitive, must start with /)
- Ensure both IAM and container policies allow mediastore-data:GetObject
- Confirm the container endpoint is correct for the target container

SHOULD:
- Use describe-object to verify the object exists before get-object
- Check content-type metadata matches expected media type

MAY:
- Implement client-side caching to reduce repeated get requests

## Safety Ratings

```
safety_ratings:
  - "mediastore-data describe-object: GREEN — read-only metadata query"
  - "mediastore-data list-items: GREEN — read-only object listing"
  - "mediastore-data get-object: GREEN — read-only data retrieval"
  - "cloudwatch logs get-log-events: GREEN — read-only log retrieval"
```

## Escalation Conditions

- Objects exist but get-object returns 500 errors consistently
- Latency for get-object exceeds SLA requirements for live streaming
- Objects are disappearing unexpectedly (check lifecycle policies)
- Container endpoint becomes unreachable

## Data Sensitivity

- Object metadata: LOW — content type, size, last modified
- Object content: HIGH — may contain live video or media content
- Access logs: MEDIUM — contain requester IPs and paths
- Object paths: LOW — file path structure

## Prohibited Actions

- NEVER assume an object exists without checking — there is no versioning
- NEVER use S3 get-object commands for MediaStore objects
- NEVER expose container endpoints with GetObject permissions to untrusted clients
- NEVER ignore 404 errors in live streaming — they indicate content gaps

## Phase 3 — Rollback

1. If objects were deleted in error, they cannot be recovered (no versioning)
2. If container policy was changed, restore previous policy
3. If IAM permissions were modified, revert to previous configuration
4. Verify live streaming playback is not affected
5. Document the retrieval failure and any data loss

## Common Issues

- symptoms: "GetObject returns 404 Not Found"
  diagnosis: "Object path is incorrect (case-sensitive) or object was deleted by lifecycle policy."
  resolution: "Verify exact path with list-items. Check lifecycle policy retention period."

- symptoms: "GetObject returns 403 Forbidden from CloudFront"
  diagnosis: "Container policy does not allow access from the CloudFront distribution."
  resolution: "Update container policy to allow the CloudFront service principal or use a permissive read policy."

## Output Format

```yaml
root_cause: "get_object — <specific_cause>"
evidence:
  - type: retrieval_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix path or permissions"
  long_term: "Implement object existence checks in application"
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

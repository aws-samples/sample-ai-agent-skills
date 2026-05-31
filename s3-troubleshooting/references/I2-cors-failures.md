---
title: "I2 — CORS Failures"
description: "Diagnose S3 CORS configuration issues for browser-based access"
status: active
severity: HIGH
triggers:
  - "CORS error"
  - "Access-Control-Allow-Origin"
  - "Preflight request"
  - "Cross-origin"
  - "CORS configuration"
owner: devops-agent
objective: "Fix S3 CORS configuration to allow browser-based cross-origin requests"
context: "CORS (Cross-Origin Resource Sharing) is required when a browser makes requests to S3 from a different domain. S3 must have a CORS configuration that allows the requesting origin, methods, and headers. Browsers send a preflight OPTIONS request first. CORS headers are only returned when the request includes an Origin header."
---

## Phase 1 — Triage

MUST:
- Check CORS configuration: `aws s3api get-bucket-cors --bucket <bucket>`
- Verify the requesting origin is in AllowedOrigins
- Verify the HTTP method is in AllowedMethods
- Check if the request includes custom headers that need to be in AllowedHeaders

SHOULD:
- Test with curl including Origin header: `curl -H "Origin: https://example.com" -H "Access-Control-Request-Method: GET" -X OPTIONS https://<bucket>.s3.<region>.amazonaws.com/<key>`
- Check browser developer tools Network tab for the preflight request and response
- Verify the S3 endpoint being used (REST API vs website endpoint)

MAY:
- Check if CloudFront is stripping or caching CORS headers
- Verify the Vary header is set correctly for caching

## Phase 2 — Remediate

MUST:
- Set CORS configuration: `aws s3api put-bucket-cors --bucket <bucket> --cors-configuration '{"CORSRules":[{"AllowedOrigins":["https://example.com"],"AllowedMethods":["GET","PUT","POST"],"AllowedHeaders":["*"],"ExposeHeaders":["ETag"],"MaxAgeSeconds":3600}]}'`
- Include all required origins, methods, and headers
- Use specific origins instead of "*" for security

SHOULD:
- Set MaxAgeSeconds to cache preflight responses and reduce OPTIONS requests
- Add ExposeHeaders for headers the browser needs to read (ETag, Content-Length)
- If using CloudFront: forward Origin, Access-Control-Request-Method, Access-Control-Request-Headers

MAY:
- Use multiple CORS rules for different origin/method combinations
- Set up CloudFront with proper cache behavior for OPTIONS requests

## Common Issues

- symptoms: "CORS error in browser but curl works fine"
  diagnosis: "curl does not enforce CORS. The browser blocks the response because S3 is not returning CORS headers."
  resolution: "Add CORS configuration to the bucket. Ensure the Origin header matches AllowedOrigins."

- symptoms: "CORS works for GET but not PUT"
  diagnosis: "PUT is not in AllowedMethods."
  resolution: "Add PUT to AllowedMethods in the CORS configuration."

- symptoms: "CORS works intermittently with CloudFront"
  diagnosis: "CloudFront caches responses without CORS headers for non-CORS requests."
  resolution: "Forward the Origin header in CloudFront cache behavior and add Vary: Origin."

## Output Format

```yaml
root_cause: "cors_failure — <specific_cause>"
evidence:
  - type: cors_config
    content: "<CORS configuration>"
  - type: request_details
    content: "<origin, method, headers>"
severity: HIGH
mitigation:
  immediate: "Fix CORS configuration to allow the requesting origin and methods"
  long_term: "Use specific origins, set MaxAgeSeconds, and configure CloudFront forwarding"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves modifying CORS configuration via put-bucket-cors. CORS changes control which origins can access bucket resources from browsers. Incorrect CORS can expose data to unintended origins. Uses get-bucket-cors for diagnosis. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- CORS changes affect browser-based access patterns for production applications

## Rollback
- Pre-change: "Save current bucket policy/ACL/CORS before modification"
- Verification: "Test access with the specific operation after change"
- Revert: "Restore previous configuration if change causes unintended access"

## Data Sensitivity
- HIGH: "Bucket policies reveal all authorized principals"
- HIGH: "ACLs expose cross-account grants"
- MEDIUM: "CORS configuration reveals allowed origins and access patterns"
- LOW: "Bucket metrics and storage class distribution"

## Prohibited Actions
- NEVER suggest disabling S3 Block Public Access as a remediation
- NEVER suggest `"Principal": "*"` without restrictive Condition keys
- NEVER suggest removing bucket encryption
- NEVER suggest `s3:*` in any policy fix
- NEVER suggest deleting a bucket to resolve configuration issues

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback

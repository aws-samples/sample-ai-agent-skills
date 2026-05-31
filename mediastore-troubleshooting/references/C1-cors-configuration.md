---
title: "C1 — CORS Configuration"
description: "Diagnose MediaStore CORS policy configuration issues"
status: active
severity: MEDIUM
triggers: ["cors error", "cross-origin blocked", "cors policy", "preflight failed", "access-control-allow-origin"]
owner: devops-agent
objective: "Resolve MediaStore CORS configuration issues for browser-based media playback"
context: "CORS policies are set at the container level and apply to all objects. They control browser-based cross-origin access to media content."
---

## Phase 1 — Triage

MUST:
- Get current CORS policy: `aws mediastore get-cors-policy --container-name <name>`
- Verify the requesting origin is included in AllowedOrigins
- Check if the HTTP method is in AllowedMethods

SHOULD:
- Test CORS with a preflight request: `curl -H "Origin: https://example.com" -H "Access-Control-Request-Method: GET" -X OPTIONS <container-endpoint>/<path>`
- Verify AllowedHeaders includes required headers (Range, Content-Type)

MAY:
- Check browser developer tools for specific CORS error messages
- Review the MaxAgeSeconds setting for preflight caching

## Phase 2 — Remediate

MUST:
- Set CORS policy: `aws mediastore put-cors-policy --container-name <name> --cors-policy file://cors.json`
- Include the correct AllowedOrigins (exact origins or wildcard *)
- Include required AllowedMethods (GET for playback, PUT for upload)

SHOULD:
- Include AllowedHeaders: ["Range", "Content-Type", "Authorization"]
- Set ExposeHeaders: ["Content-Length", "Content-Range", "ETag"]
- Set MaxAgeSeconds for preflight caching (e.g., 3600)

MAY:
- Use wildcard origin (*) for public media content only

## Safety Ratings

```
safety_ratings:
  - "mediastore get-cors-policy: GREEN — read-only CORS policy retrieval"
  - "mediastore put-cors-policy: YELLOW — changes cross-origin access control"
  - "mediastore delete-cors-policy: RED — removes all CORS configuration"
  - "curl OPTIONS preflight: GREEN — read-only CORS test"
```

## Escalation Conditions

- CORS policy is correct but browser still blocks requests (CDN caching issue)
- Wildcard origin is required but security policy prohibits it
- CORS changes do not take effect immediately (CDN cache invalidation needed)
- Multiple origins need different CORS rules (not supported per-path)

## Data Sensitivity

- CORS policy document: LOW — contains allowed origins and methods
- Allowed origins list: LOW — domain names
- Preflight response headers: LOW — CORS headers
- Browser error messages: LOW — CORS violation details

## Prohibited Actions

- NEVER use wildcard origin (*) for containers with sensitive or paid content
- NEVER delete the CORS policy without understanding the impact on browser clients
- NEVER include credentials (AllowCredentials) with wildcard origins
- NEVER ignore CORS errors as "browser issues" — they indicate real configuration problems

## Phase 3 — Rollback

1. Restore previous CORS policy: `aws mediastore put-cors-policy --container-name <name> --cors-policy file://previous-cors.json`
2. If CORS was deleted, re-apply: `aws mediastore put-cors-policy --container-name <name> --cors-policy file://cors.json`
3. Invalidate CDN cache if CloudFront is caching CORS responses
4. Verify browser-based playback is restored
5. Document the CORS configuration change

## Common Issues

- symptoms: "Browser shows 'No Access-Control-Allow-Origin header' error"
  diagnosis: "CORS policy is not set on the container or the requesting origin is not in AllowedOrigins."
  resolution: "Set CORS policy with the correct AllowedOrigins. Include the exact origin URL."

- symptoms: "CORS works for GET but fails for PUT requests"
  diagnosis: "AllowedMethods does not include PUT."
  resolution: "Add PUT to AllowedMethods in the CORS policy."

## Output Format

```yaml
root_cause: "cors — <specific_cause>"
evidence:
  - type: cors_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix CORS policy configuration"
  long_term: "Include CORS policy in IaC templates"
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

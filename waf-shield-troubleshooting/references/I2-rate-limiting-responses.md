---
title: "I2 — Rate Limiting Response Issues"
description: "Diagnose custom response configuration for rate-limited requests"
status: active
severity: MEDIUM
triggers:
  - "rate limit response"
  - "429 response"
  - "rate limit message"
  - "throttle response"
  - "retry-after"
owner: devops-agent
objective: "Configure proper responses for rate-limited requests"
context: "Rate-based rules can use custom responses to return meaningful error messages to rate-limited clients. Best practice is to return HTTP 429 (Too Many Requests) with a Retry-After header. Custom response bodies can include rate limit information in JSON or HTML format. Rate-based rules support Block action with custom responses, or can use CAPTCHA/Challenge actions."
---

## Phase 1 — Triage

MUST:
- Check the rate-based rule action and custom response: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id> --query 'WebACL.Rules[?Statement.RateBasedStatement].{Name:Name,Action:Action}'`
- Verify if a custom response is configured for the rate-based rule
- Check the response status code (should be 429 for rate limiting)
- Verify custom response headers (Retry-After)

SHOULD:
- Check if the custom response body is defined in the web ACL
- Verify the response format matches client expectations (JSON for APIs)
- Test the rate limit response by exceeding the threshold

MAY:
- Check if different rate-based rules need different response formats
- Review client-side handling of the rate limit response

## Phase 2 — Remediate

MUST:
- Configure the rate-based rule with a custom response: set CustomResponse with ResponseCode 429
- Define a custom response body with rate limit information
- Add Retry-After header to the custom response

SHOULD:
- Use JSON format for API endpoints: `{"error": "rate_limit_exceeded", "retry_after": 300}`
- Use HTML format for web endpoints with a user-friendly message
- Include the rate limit window information in the response

MAY:
- Implement different response formats for different paths (API vs web)
- Add custom headers with rate limit details (X-RateLimit-Limit, X-RateLimit-Remaining)
- Use CAPTCHA action instead of Block for interactive users

## Common Issues

- symptoms: "Rate-limited clients receive generic 403 instead of 429"
  diagnosis: "No custom response configured on the rate-based rule."
  resolution: "Add CustomResponse with ResponseCode 429 and a custom body."

- symptoms: "API clients not handling rate limit response correctly"
  diagnosis: "Response format is HTML but API expects JSON."
  resolution: "Set content type to application/json and provide a JSON response body."

- symptoms: "Retry-After header not present in rate limit response"
  diagnosis: "Custom response headers not configured."
  resolution: "Add ResponseHeaders with Retry-After set to the rate limit window (e.g., 300 seconds)."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Configure 429 response | YELLOW | Rule modification; reversible |
| Add Retry-After header | YELLOW | Configuration change; reversible |
| Define custom response body | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl` (rate rule actions) | LOW | Rule configuration |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Custom response configuration | Revert via `update-web-acl` |
| Response header addition | Remove headers via `update-web-acl` |

## Output Format

```yaml
root_cause: "rate_limiting_response — <specific_cause>"
evidence:
  - type: rule_action
    content: "<rate-based rule action and custom response>"
  - type: response_code
    content: "<HTTP status code returned>"
  - type: response_body
    content: "<custom response body content>"
severity: MEDIUM
mitigation:
  immediate: "Configure 429 response with Retry-After header"
  long_term: "Implement format-appropriate responses for all rate-limited paths"
```

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
  - "NEVER suggest disabling WAF to fix access issues"
  - "NEVER suggest removing all WAF rules"
  - "NEVER suggest allowing all IPs to bypass rate limiting"

---
title: "I1 — Custom Response Body Issues"
description: "Diagnose custom response body configuration and delivery problems"
status: active
severity: MEDIUM
triggers:
  - "custom response"
  - "custom error page"
  - "WAF response body"
  - "block response"
  - "custom 403"
owner: devops-agent
objective: "Identify and fix custom response body configuration issues"
context: "WAF custom response bodies allow you to return custom content when blocking requests. Bodies are limited to 4 KB each, with a maximum of 50 per web ACL. Supported content types are application/json, text/html, and text/plain. Custom responses can include custom headers (up to 5 per action). Custom responses only apply to Block actions. The response status code can be 200-599."
---

## Phase 1 — Triage

MUST:
- Check custom response bodies in the web ACL: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id> --query 'WebACL.CustomResponseBodies'`
- Check which rules use custom responses: look for `CustomResponse` in rule Block actions
- Verify the response body key matches a defined custom response body
- Check the content type and response status code

SHOULD:
- Verify custom response headers are correctly configured
- Test the custom response by triggering the blocking rule
- Check if the custom response body size is within the 4 KB limit

MAY:
- Review the custom response rendering in different browsers/clients
- Check if API clients handle the custom response format correctly

## Phase 2 — Remediate

MUST:
- Define custom response bodies in the web ACL configuration
- Reference the correct body key in the rule's Block action CustomResponse
- Set the appropriate content type (application/json for APIs, text/html for web)
- Keep body size under 4 KB

SHOULD:
- Use different custom responses for different rule types (API errors vs web pages)
- Include useful information in the response (request ID, support contact)
- Set appropriate HTTP status codes (403 for blocked, 429 for rate-limited)

MAY:
- Implement localized custom responses using custom headers
- Create branded error pages for web applications
- Include retry-after headers for rate-limited responses

## Common Issues

- symptoms: "Custom response body not returned — default WAF 403 shown"
  diagnosis: "CustomResponse body key does not match any defined CustomResponseBodies key."
  resolution: "Verify the body key in the rule action matches a key in WebACL.CustomResponseBodies."

- symptoms: "Custom response body truncated"
  diagnosis: "Body exceeds the 4 KB limit."
  resolution: "Reduce the response body size to under 4 KB. Minimize HTML/JSON content."

- symptoms: "Custom response not applied to managed rule group blocks"
  diagnosis: "Custom responses can only be set on rules you control, not directly on managed rule group internal rules."
  resolution: "Override managed rules to Count, then create a custom rule matching the label with a Block action and custom response."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Define custom response bodies | YELLOW | Configuration change; reversible |
| Set custom response on rules | YELLOW | Rule modification; reversible |
| Override managed rules for custom response | YELLOW | Rule behavior change; reversible |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl` (custom responses) | LOW | Response body configuration |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Custom response body definition | Remove via `update-web-acl` |
| Custom response on rule | Remove CustomResponse from rule action |
| Managed rule override | Revert override via `update-web-acl` |

## Output Format

```yaml
root_cause: "custom_response — <specific_cause>"
evidence:
  - type: response_bodies
    content: "<defined custom response bodies>"
  - type: rule_config
    content: "<rule action with custom response reference>"
  - type: content_type
    content: "<response content type>"
severity: MEDIUM
mitigation:
  immediate: "Fix body key reference or content size"
  long_term: "Implement comprehensive custom responses for all blocking scenarios"
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

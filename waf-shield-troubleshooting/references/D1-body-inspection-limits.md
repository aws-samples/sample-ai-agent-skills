---
title: "D1 — Body Inspection Limits"
description: "Diagnose issues where WAF does not inspect the full request body"
status: active
severity: HIGH
triggers:
  - "body inspection"
  - "request body"
  - "body size limit"
  - "payload not inspected"
  - "oversize handling"
owner: devops-agent
objective: "Identify and resolve body inspection limit issues that allow uninspected content through WAF"
context: "WAF inspects only the first portion of the request body. The default limit is 8 KB. For REGIONAL web ACLs, this can be increased to 16 KB, 32 KB, or 64 KB. For CLOUDFRONT web ACLs, the limit is fixed at 8 KB. Content beyond the limit is handled by the oversize handling action: Continue (skip inspection), Match (treat as matching), or No Match (treat as not matching)."
---

## Phase 1 — Triage

MUST:
- Check the current body inspection limit: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id> --query 'WebACL.Rules[*].Statement.*.FieldToMatch.Body'`
- Check the oversize handling configuration for body-inspecting rules
- Determine the typical request body size for the application
- Verify the web ACL scope (CLOUDFRONT is fixed at 8 KB)

SHOULD:
- Check WAF logs for requests with large bodies that were allowed
- Review which rules inspect the body (SQLi, XSS, custom string match)
- Check the WCU impact of increasing the body inspection limit

MAY:
- Analyze application traffic to determine the distribution of request body sizes
- Test with payloads at and beyond the inspection limit

## Phase 2 — Remediate

MUST:
- Increase the body inspection limit for REGIONAL web ACLs if needed: configure SizeInspectionLimit in the rule's FieldToMatch Body configuration
- Set appropriate oversize handling: Match (block oversized requests) for security-critical paths, Continue for paths where large bodies are expected
- For CLOUDFRONT web ACLs, implement application-level body validation for content beyond 8 KB

SHOULD:
- Use different oversize handling for different rules based on the path
- Monitor WCU consumption after increasing the body inspection limit
- Add size constraint rules to reject unexpectedly large request bodies

MAY:
- Implement API Gateway request validation as an additional layer
- Use Lambda@Edge or CloudFront Functions for pre-WAF body size checks

## Common Issues

- symptoms: "SQL injection in POST body not detected by WAF"
  diagnosis: "Injection payload is beyond the 8 KB default body inspection limit."
  resolution: "Increase body inspection limit to 16/32/64 KB for REGIONAL. For CLOUDFRONT, add application-level validation."

- symptoms: "File upload requests being blocked"
  diagnosis: "Oversize handling is set to Match, treating all large bodies as matching the rule."
  resolution: "Set oversize handling to Continue for upload paths, or exclude upload paths with scope-down."

- symptoms: "Increased WCU consumption after changing body inspection"
  diagnosis: "Larger body inspection limits consume more WCUs per rule."
  resolution: "Optimize other rules to free WCU capacity. Request WCU limit increase if needed."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Increase body inspection limit | YELLOW | Configuration change; reversible |
| Set oversize handling | YELLOW | Rule behavior change; reversible |
| Add size constraint rules | YELLOW | New rule; can be removed |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- Body inspection limit changes affecting WCU consumption

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl` (body config) | LOW | Rule configuration |
| WAF logs | HIGH | Request headers and metadata |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Body inspection limit increase | Revert limit via `update-web-acl` |
| Oversize handling change | Revert via `update-web-acl` |
| Size constraint rule addition | Remove rule via `update-web-acl` |

## Output Format

```yaml
root_cause: "body_inspection — <specific_cause>"
evidence:
  - type: inspection_limit
    content: "<current body inspection limit>"
  - type: oversize_handling
    content: "<Continue, Match, or NoMatch>"
  - type: request_body_size
    content: "<typical body size for the application>"
severity: HIGH
mitigation:
  immediate: "Increase body inspection limit or adjust oversize handling"
  long_term: "Implement layered body validation with appropriate limits per path"
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

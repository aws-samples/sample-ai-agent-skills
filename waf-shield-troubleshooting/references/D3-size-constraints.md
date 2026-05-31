---
title: "D3 — Size Constraint Issues"
description: "Diagnose size constraint statement problems in WAF rules"
status: active
severity: MEDIUM
triggers:
  - "size constraint"
  - "request size"
  - "body size"
  - "header size"
  - "query string size"
owner: devops-agent
objective: "Identify and fix size constraint rule configuration issues"
context: "WAF size constraint statements compare the size of a request component (body, URI, query string, headers, cookies) against a threshold. Comparison operators include EQ, NE, LE, LT, GE, GT. Size is measured in bytes. Size constraints are useful for blocking oversized requests that may indicate attacks. They consume 1 WCU each."
---

## Phase 1 — Triage

MUST:
- Find size constraint rules in the web ACL: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id> --query 'WebACL.Rules[*].{Name:Name,Priority:Priority,Statement:Statement}'`
- Check the comparison operator and size threshold
- Identify which request component is being measured
- Check sampled requests to see actual sizes of blocked/allowed requests

SHOULD:
- Verify the size threshold is appropriate for the application's normal traffic
- Check if text transformations affect the measured size
- Review if the size constraint interacts with body inspection limits

MAY:
- Analyze application traffic to determine normal size distributions
- Test with requests of various sizes to verify the constraint behavior

## Phase 2 — Remediate

MUST:
- Set appropriate size thresholds based on application requirements
- Use the correct comparison operator (GT to block requests larger than threshold)
- Consider the interaction between size constraints and body inspection limits

SHOULD:
- Use different size constraints for different paths (API vs file upload)
- Combine size constraints with other conditions using AND statements
- Test size constraint changes in Count mode first

MAY:
- Implement graduated size limits (warn at one threshold, block at another)
- Use API Gateway request validation for additional size enforcement

## Common Issues

- symptoms: "Legitimate file uploads blocked by size constraint"
  diagnosis: "Size constraint threshold is too low for the upload path."
  resolution: "Increase threshold or add scope-down to exclude upload paths."

- symptoms: "Size constraint not blocking oversized requests"
  diagnosis: "Comparison operator is wrong (e.g., LT instead of GT) or threshold is too high."
  resolution: "Fix the comparison operator. Use GT with the maximum allowed size."

- symptoms: "Size constraint on body not working for large payloads"
  diagnosis: "Body inspection limit (8 KB default) is smaller than the size threshold."
  resolution: "Increase body inspection limit to match or exceed the size constraint threshold."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Adjust size threshold | YELLOW | Rule modification; reversible |
| Fix comparison operator | YELLOW | Rule modification; reversible |
| Add scope-down statements | YELLOW | Rule modification; reversible |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- Size constraint changes affecting file uploads or API traffic

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl` (size constraints) | LOW | Rule configuration |
| `get-sampled-requests` | MEDIUM | Request sizes and headers |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Threshold change | Revert threshold via `update-web-acl` |
| Operator change | Revert operator via `update-web-acl` |
| Scope-down addition | Remove scope-down via `update-web-acl` |

## Output Format

```yaml
root_cause: "size_constraint — <specific_cause>"
evidence:
  - type: constraint_config
    content: "<component, operator, threshold>"
  - type: actual_sizes
    content: "<sizes from sampled requests>"
  - type: body_inspection_limit
    content: "<current limit>"
severity: MEDIUM
mitigation:
  immediate: "Adjust size threshold or comparison operator"
  long_term: "Implement path-specific size constraints with scope-down statements"
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

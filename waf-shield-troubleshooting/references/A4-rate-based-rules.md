---
title: "A4 — WAF Rate-Based Rule Issues"
description: "Diagnose rate-based rules not triggering, over-blocking, or behaving unexpectedly"
status: active
severity: HIGH
triggers:
  - "rate limit"
  - "rate-based rule"
  - "throttling"
  - "too many requests"
  - "rate blocking"
owner: devops-agent
objective: "Identify why rate-based rules are not working as expected and configure proper rate limiting"
context: "Rate-based rules track request rates over a sliding 5-minute window. The minimum threshold is 100 requests. When a tracked key (IP, forwarded IP, custom key) exceeds the threshold, the rule action applies. There is a short delay (30s-2min) between exceeding the threshold and enforcement. IPs are automatically unblocked when their rate drops below the threshold."
---

## Phase 1 — Triage

MUST:
- Get the rate-based rule configuration: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id> --query 'WebACL.Rules[?Statement.RateBasedStatement]'`
- Check currently rate-limited IPs: `aws wafv2 get-rate-based-statement-managed-keys --scope <scope> --web-acl-name <acl-name> --web-acl-id <acl-id> --rule-name <rule-name>`
- Check CloudWatch metrics for the rate-based rule: `aws cloudwatch get-metric-statistics --namespace AWS/WAFV2 --metric-name BlockedRequests --dimensions Name=WebACL,Value=<acl-name> Name=Rule,Value=<rule-name> Name=Region,Value=<region> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify the aggregation key type (IP, FORWARDED_IP, custom keys)

SHOULD:
- Check if a higher-priority Allow rule is matching before the rate-based rule
- Verify the threshold is appropriate for the traffic pattern
- Check if the rule has a scope-down statement that may be too restrictive

MAY:
- Review WAF logs for request patterns from the source IP
- Check if the client is behind a CDN or proxy (X-Forwarded-For header)

## Phase 2 — Remediate

MUST:
- Set threshold to at least 100 (minimum allowed)
- Use FORWARDED_IP aggregation if clients are behind a load balancer or CDN, and specify the correct header (X-Forwarded-For)
- Verify the rate-based rule priority allows it to evaluate (not short-circuited by earlier Allow rules)

SHOULD:
- Add scope-down statements to rate-limit only specific paths (e.g., /login, /api)
- Use custom keys (header values, query strings) for more granular rate limiting
- Combine rate-based rules with IP reputation rules for defense in depth

MAY:
- Create multiple rate-based rules with different thresholds for different paths
- Use the rate-based rule in Count mode with labels, then add a separate Block rule matching the label

## Common Issues

- symptoms: "Rate-based rule not blocking despite high request volume"
  diagnosis: "Aggregation key is IP but clients are behind a proxy — all requests appear from the proxy IP, or threshold is too high."
  resolution: "Switch to FORWARDED_IP aggregation with the correct header name."

- symptoms: "Legitimate users being rate-limited"
  diagnosis: "Multiple users share a single IP (corporate NAT, mobile carrier). Threshold is too low for shared IPs."
  resolution: "Increase threshold or use custom aggregation keys (e.g., session token header)."

- symptoms: "Rate-based rule blocks but attacker switches IPs"
  diagnosis: "Attacker uses distributed IPs to stay below per-IP threshold."
  resolution: "Add IP reputation managed rules (AWSManagedRulesAmazonIpReputationList) and Bot Control."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Adjust rate threshold | YELLOW | Threshold change; reversible |
| Switch to FORWARDED_IP aggregation | YELLOW | Aggregation change; reversible |
| Add scope-down statements | YELLOW | Rule modification; reversible |
| Add custom aggregation keys | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- Rate limit threshold changes affecting production traffic

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl` (rate rules) | LOW | Rule configuration |
| `get-rate-based-statement-managed-keys` | MEDIUM | Currently rate-limited IPs |
| CloudWatch metrics | LOW | Aggregate request counts |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting
- NEVER set rate threshold below 100 (minimum allowed)

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Threshold change | Revert threshold via `update-web-acl` |
| Aggregation key change | Revert to previous aggregation via `update-web-acl` |
| Scope-down addition | Remove scope-down via `update-web-acl` |

## Output Format

```yaml
root_cause: "rate_based_rule — <specific_cause>"
evidence:
  - type: rule_config
    content: "<threshold, aggregation key, scope-down>"
  - type: managed_keys
    content: "<currently rate-limited IPs>"
  - type: cloudwatch_metrics
    content: "<blocked request count over time>"
severity: HIGH
mitigation:
  immediate: "Adjust threshold or aggregation key"
  long_term: "Implement multi-layer rate limiting with custom keys and IP reputation"
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

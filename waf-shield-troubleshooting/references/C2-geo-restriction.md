---
title: "C2 — Geo-Restriction Issues"
description: "Diagnose geographic-based blocking or allowing not working as expected"
status: active
severity: MEDIUM
triggers:
  - "geo restriction"
  - "geo blocking"
  - "country block"
  - "geographic"
  - "country filter"
owner: devops-agent
objective: "Identify and fix geo-restriction rule configuration issues"
context: "WAF geo match statements match requests based on the country of origin determined by the source IP address. WAF uses a GeoIP database that may not be 100% accurate, especially for VPN/proxy users. Geo match can be combined with other conditions using AND/OR logic. Country codes use ISO 3166-1 alpha-2 format."
---

## Phase 1 — Triage

MUST:
- Find the geo match rule in the web ACL: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id> --query 'WebACL.Rules[?Statement.GeoMatchStatement || Statement.AndStatement || Statement.OrStatement]'`
- Verify the country codes are correct (ISO 3166-1 alpha-2, e.g., US, GB, DE)
- Check the rule action and priority
- Verify the source IP is not a VPN/proxy that resolves to a different country

SHOULD:
- Check if ForwardedIPConfig is needed (clients behind proxies)
- Verify the geo match is not negated unintentionally (NotStatement)
- Check sampled requests to see the country code WAF resolved

MAY:
- Test from different geographic locations using VPN or cloud instances
- Check WAF logs for the country field in request details

## Phase 2 — Remediate

MUST:
- Use correct ISO 3166-1 alpha-2 country codes in the geo match statement
- Configure ForwardedIPConfig if clients are behind a CDN or proxy
- Set appropriate rule priority relative to other rules

SHOULD:
- Combine geo match with other conditions (e.g., block country X except for specific IPs)
- Use Count mode first to verify geo matching accuracy before blocking
- Document the business reason for each geo-restriction

MAY:
- Use CloudFront geographic restrictions for CloudFront-specific geo-blocking (separate from WAF)
- Implement geo-based rate limiting instead of outright blocking

## Common Issues

- symptoms: "Users from blocked country can still access the site"
  diagnosis: "Users are using VPN/proxy with exit nodes in allowed countries."
  resolution: "Combine geo-restriction with IP reputation rules and Bot Control."

- symptoms: "Legitimate users from allowed country are blocked"
  diagnosis: "GeoIP database maps their IP to a different country (ISP routing, mobile carrier)."
  resolution: "Add specific IP ranges to an allow list with higher priority than the geo block."

- symptoms: "Geo match rule not matching any requests"
  diagnosis: "Country codes are incorrect or the rule has a scope-down statement that excludes all traffic."
  resolution: "Verify country codes. Check scope-down statements. Test in Count mode."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix country codes | YELLOW | Rule modification; reversible |
| Configure ForwardedIPConfig | YELLOW | Rule modification; reversible |
| Use Count mode for testing | YELLOW | Adds Count override; reversible |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- Geo-restriction changes affecting production traffic

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl` (geo rules) | LOW | Rule configuration |
| `get-sampled-requests` (country) | MEDIUM | Source country and IP |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Country code change | Revert country codes via `update-web-acl` |
| ForwardedIPConfig change | Revert via `update-web-acl` |
| Count mode switch | Revert to Block via `update-web-acl` |

## Output Format

```yaml
root_cause: "geo_restriction — <specific_cause>"
evidence:
  - type: geo_rule
    content: "<country codes and action>"
  - type: source_country
    content: "<country WAF resolved for the IP>"
  - type: rule_priority
    content: "<priority relative to other rules>"
severity: MEDIUM
mitigation:
  immediate: "Fix country codes or forwarded IP configuration"
  long_term: "Combine geo-restriction with IP reputation and allow list exceptions"
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

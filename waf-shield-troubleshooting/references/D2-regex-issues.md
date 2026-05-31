---
title: "D2 — Regex Pattern Issues"
description: "Diagnose regex pattern matching failures and regex pattern set problems"
status: active
severity: MEDIUM
triggers:
  - "regex"
  - "regex pattern"
  - "pattern not matching"
  - "regex set"
  - "regular expression"
owner: devops-agent
objective: "Identify and fix regex pattern matching issues in WAF rules"
context: "WAF supports regex pattern sets for flexible string matching. Each set can contain up to 10 regex patterns, each up to 200 characters. WAF uses a Java-compatible regex syntax. Regex rules consume 25 WCUs per pattern set. Complex patterns may cause performance issues. Regex match statements can inspect URI, query string, headers, body, and other request components."
---

## Phase 1 — Triage

MUST:
- Get the regex pattern set: `aws wafv2 get-regex-pattern-set --name <set-name> --scope <scope> --id <set-id>`
- Verify the regex syntax is valid and matches the intended patterns
- Check which request component the regex is applied to (URI, query string, body, header)
- Verify text transformations are applied before regex matching

SHOULD:
- Test the regex pattern against sample request data
- Check the WCU consumption of regex rules (25 WCUs per pattern set)
- Verify the regex pattern set scope matches the web ACL scope

MAY:
- Use an online regex tester with Java/PCRE syntax to validate patterns
- Check WAF logs for requests that should have matched but didn't

## Phase 2 — Remediate

MUST:
- Fix regex syntax errors in the pattern set: `aws wafv2 update-regex-pattern-set --name <set-name> --scope <scope> --id <set-id> --regular-expression-list RegexString=<pattern> --lock-token <lock-token>`
- Apply appropriate text transformations (URL_DECODE, LOWERCASE) before regex matching
- Stay within the 10 patterns per set and 200 characters per pattern limits

SHOULD:
- Simplify complex regex patterns to reduce WCU consumption and improve performance
- Use string match statements instead of regex when exact or prefix matching suffices
- Test regex changes in Count mode before enforcing

MAY:
- Split complex patterns across multiple regex pattern sets
- Use byte match statements for simple pattern matching (lower WCU cost)

## Common Issues

- symptoms: "Regex pattern not matching URL-encoded characters"
  diagnosis: "Text transformation URL_DECODE is not applied before regex matching."
  resolution: "Add URL_DECODE text transformation to the rule statement."

- symptoms: "Regex pattern set update fails"
  diagnosis: "Pattern exceeds 200 character limit or set exceeds 10 patterns."
  resolution: "Simplify patterns or split across multiple pattern sets."

- symptoms: "Regex rule consuming too many WCUs"
  diagnosis: "Multiple regex pattern sets in the web ACL, each costing 25 WCUs."
  resolution: "Consolidate patterns. Use string match for simple patterns. Remove unused regex rules."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix regex patterns | YELLOW | Pattern change; reversible |
| Add text transformations | YELLOW | Rule modification; reversible |
| Simplify patterns | YELLOW | Pattern change; reversible |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- Regex pattern changes affecting traffic matching

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-regex-pattern-set` | LOW | Regex patterns |
| `get-web-acl` | LOW | Rule configuration |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Regex pattern change | Revert via `update-regex-pattern-set` with previous patterns |
| Text transformation addition | Revert via `update-web-acl` |

## Output Format

```yaml
root_cause: "regex_issues — <specific_cause>"
evidence:
  - type: regex_patterns
    content: "<patterns in the set>"
  - type: text_transformations
    content: "<transformations applied>"
  - type: match_component
    content: "<URI, query string, body, etc.>"
severity: MEDIUM
mitigation:
  immediate: "Fix regex syntax or add text transformations"
  long_term: "Optimize regex usage and prefer simpler match types where possible"
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

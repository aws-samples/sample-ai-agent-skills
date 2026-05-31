---
title: "A3 — WAF Rule Priority Issues"
description: "Diagnose rule evaluation order problems causing unexpected allow or block behavior"
status: active
severity: HIGH
triggers:
  - "rule priority"
  - "rule order"
  - "evaluation order"
  - "wrong rule matching"
  - "rule not evaluated"
owner: devops-agent
objective: "Identify and fix rule priority ordering issues that cause incorrect request handling"
context: "WAF evaluates rules by priority number in ascending order (lowest first). A terminating action (Block or Allow) stops evaluation. If an Allow rule has a lower priority number than a Block rule, matching requests are allowed before the Block rule is ever evaluated. Count actions do not terminate evaluation. The web ACL default action applies to requests that match no rules."
---

## Phase 1 — Triage

MUST:
- List all rules with their priorities and actions: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id> --query 'WebACL.Rules[*].{Name:Name,Priority:Priority,Action:Action,OverrideAction:OverrideAction,Statement:Statement}'`
- Identify the rule that is matching the request: check sampled requests: `aws wafv2 get-sampled-requests --web-acl-arn <acl-arn> --rule-metric-name <metric-name> --scope <scope> --time-window StartTime=<start>,EndTime=<end> --max-items 100`
- Verify the web ACL default action: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id> --query 'WebACL.DefaultAction'`
- Check if a higher-priority (lower number) Allow rule is matching before the intended Block rule

SHOULD:
- Map out the complete rule evaluation flow: list rules sorted by priority
- Check for overlapping match conditions between Allow and Block rules
- Verify managed rule group override actions (None vs Count)

MAY:
- Use WAF logs to trace the evaluation path for specific requests
- Check labels applied by earlier rules that may affect later rule matching

## Phase 2 — Remediate

MUST:
- Reorder rule priorities so that specific exceptions (Allow) come before broad protections (Block) only when intended
- Ensure IP allow lists have lower priority numbers than managed rule groups if trusted IPs should bypass WAF
- Verify that Count rules are positioned to add labels before label-matching rules

SHOULD:
- Document the intended rule evaluation order
- Use scope-down statements instead of priority-based exceptions where possible
- Test priority changes in a staging environment first

MAY:
- Consolidate overlapping rules to simplify the evaluation chain
- Use labels for multi-stage evaluation instead of relying solely on priority ordering

## Common Issues

- symptoms: "Block rule never triggers even though requests match its conditions"
  diagnosis: "A higher-priority Allow rule matches the same requests first, terminating evaluation."
  resolution: "Adjust priorities so the Block rule evaluates before the Allow rule, or narrow the Allow rule."

- symptoms: "All requests are allowed despite having Block rules"
  diagnosis: "All Block rules are set to Count override, or the default action is Allow and no rules match."
  resolution: "Check OverrideAction on managed rule groups. Verify rules are not all in Count mode."

- symptoms: "Label-based rule never matches"
  diagnosis: "The label-producing rule has a higher priority number (evaluated later) than the label-matching rule."
  resolution: "Ensure the label-producing rule has a lower priority number than the label-matching rule."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Reorder rule priorities | YELLOW | Evaluation order change; reversible |
| Adjust scope-down statements | YELLOW | Rule modification; reversible |
| Consolidate overlapping rules | YELLOW | Rule restructure; reversible with backup |

## Escalation Conditions

- Production web ACL rule priority changes
- Shield Advanced configuration changes
- Changes affecting the evaluation order of security rules

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl` (rules and priorities) | LOW | Rule configuration |
| `get-sampled-requests` | MEDIUM | Request headers and IPs |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Priority reorder | Revert priorities via `update-web-acl` with previous values |
| Scope-down change | Revert scope-down via `update-web-acl` |
| Rule consolidation | Restore original rules via `update-web-acl` |

## Output Format

```yaml
root_cause: "rule_priority — <specific_ordering_issue>"
evidence:
  - type: rule_list
    content: "<rules sorted by priority with actions>"
  - type: sampled_request
    content: "<request matched by unexpected rule>"
  - type: default_action
    content: "<web ACL default action>"
severity: HIGH
mitigation:
  immediate: "Reorder rule priorities to achieve intended evaluation flow"
  long_term: "Document rule ordering strategy and use scope-down statements"
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

---
title: "B1 — AWS Managed Rule Group Issues"
description: "Diagnose problems with AWS managed rule groups blocking or not blocking as expected"
status: active
severity: HIGH
triggers:
  - "managed rules"
  - "AWS managed rule"
  - "AWSManagedRules"
  - "CommonRuleSet"
  - "SQLiRuleSet"
owner: devops-agent
objective: "Identify and resolve issues with AWS managed rule group configuration and behavior"
context: "AWS managed rule groups are pre-configured rule sets maintained by AWS. They include CommonRuleSet, SQLiRuleSet, KnownBadInputsRuleSet, LinuxRuleSet, and others. Each group has a WCU cost and can be overridden to Count mode. Individual rules within a group can be overridden independently. Groups are versioned and AWS may update the default version."
---

## Phase 1 — Triage

MUST:
- List managed rule groups in the web ACL: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id> --query 'WebACL.Rules[?Statement.ManagedRuleGroupStatement]'`
- Check the override action for each managed rule group (None = use group actions, Count = override all to Count)
- Describe the managed rule group to see individual rules: `aws wafv2 describe-managed-rule-group --vendor-name AWS --name <rule-group-name> --scope <scope>`
- Check for individual rule action overrides within the group: look for `RuleActionOverrides` in the web ACL config
- Check sampled requests for the specific managed rule: `aws wafv2 get-sampled-requests --web-acl-arn <acl-arn> --rule-metric-name <metric-name> --scope <scope> --time-window StartTime=<start>,EndTime=<end> --max-items 100`

SHOULD:
- Check the current version of the managed rule group in use
- Review the WCU consumption of each managed rule group
- Check labels applied by the managed rule group for use in custom rules

MAY:
- Compare behavior between the current version and the latest available version
- Review AWS documentation for the specific rules within the group

## Phase 2 — Remediate

MUST:
- Set OverrideAction to None to enforce the managed rule group's actions (if currently Count)
- Use RuleActionOverrides to set specific problematic rules to Count while keeping others active
- Add scope-down statements to limit managed rule evaluation to relevant request paths

SHOULD:
- Pin to a specific version if automatic updates cause issues
- Subscribe to SNS notifications for managed rule group version changes
- Test new versions in Count mode before switching to Block

MAY:
- Create excluded rules lists for known false positive patterns
- Combine multiple managed rule groups for layered protection

## Common Issues

- symptoms: "Managed rule group not blocking anything"
  diagnosis: "OverrideAction is set to Count, overriding all rules in the group to Count mode."
  resolution: "Change OverrideAction to None to use the rule group's native actions."

- symptoms: "Specific managed rule causing false positives on API endpoints"
  diagnosis: "Rule matches legitimate API payload patterns (JSON, XML, encoded data)."
  resolution: "Override the specific rule to Count using RuleActionOverrides. Add scope-down to exclude API paths."

- symptoms: "Managed rule group consuming too many WCUs"
  diagnosis: "Multiple managed rule groups combined exceed the 1,500 WCU limit."
  resolution: "Check WCU per group with describe-managed-rule-group. Remove less critical groups or request limit increase."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Set OverrideAction to None | YELLOW | Enables blocking; may cause false positives |
| Add RuleActionOverrides | YELLOW | Per-rule override; reversible |
| Add scope-down statements | YELLOW | Rule modification; reversible |
| Pin to specific version | YELLOW | Version lock; reversible |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- Switching managed rules from Count to Block in production

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl` (managed rules) | LOW | Rule configuration |
| `describe-managed-rule-group` | LOW | Rule group metadata |
| `get-sampled-requests` | MEDIUM | Request headers and IPs |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting
- NEVER remove all managed rule groups simultaneously

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| OverrideAction change | Revert to Count via `update-web-acl` |
| RuleActionOverrides | Remove overrides via `update-web-acl` |
| Scope-down addition | Remove scope-down via `update-web-acl` |
| Version pin | Unpin (set version to null) via `update-web-acl` |

## Output Format

```yaml
root_cause: "managed_rules — <specific_cause>"
evidence:
  - type: override_action
    content: "<None or Count>"
  - type: rule_action_overrides
    content: "<individually overridden rules>"
  - type: managed_group_version
    content: "<current version in use>"
severity: HIGH
mitigation:
  immediate: "Fix override action or individual rule overrides"
  long_term: "Implement version pinning and scope-down statements"
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

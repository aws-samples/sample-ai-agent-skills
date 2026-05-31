---
title: "A1 — WAF False Positives"
description: "Diagnose and resolve WAF rules incorrectly blocking legitimate traffic"
status: active
severity: HIGH
triggers:
  - "false positive"
  - "legitimate traffic blocked"
  - "WAF blocking valid requests"
  - "403 from WAF"
  - "request blocked incorrectly"
owner: devops-agent
objective: "Identify which WAF rule is incorrectly blocking legitimate requests and remediate without reducing security posture"
context: "False positives occur when WAF rules match patterns in legitimate requests. Common causes include overly broad managed rule groups, SQL-like syntax in form fields, XML/JSON payloads triggering XSS rules, and API tokens matching injection patterns. Sampled requests and WAF logs are the primary diagnostic tools."
---

## Phase 1 — Triage

MUST:
- Identify the blocking rule from sampled requests: `aws wafv2 get-sampled-requests --web-acl-arn <acl-arn> --rule-metric-name <metric-name> --scope <REGIONAL|CLOUDFRONT> --time-window StartTime=<start>,EndTime=<end> --max-items 100`
- Check WAF logs for the blocked request details (terminatingRuleId, labels, request headers)
- Get the web ACL configuration to see all rules and priorities: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id>`
- If a managed rule group is blocking, identify the specific rule: `aws wafv2 describe-managed-rule-group --vendor-name AWS --name <rule-group-name> --scope <scope>`

SHOULD:
- Check CloudWatch metrics per rule to quantify the false positive rate: `aws cloudwatch get-metric-statistics --namespace AWS/WAFV2 --metric-name BlockedRequests --dimensions Name=WebACL,Value=<acl-name> Name=Rule,Value=<rule-name> Name=Region,Value=<region> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Review the request URI, headers, query string, and body that triggered the block
- Check if labels were applied by earlier rules that influenced the blocking rule

MAY:
- Reproduce the false positive with a test request: `curl -v -H "X-Test: true" https://<domain>/<path>`
- Check WAF logs in the logging destination for full request details

## Phase 2 — Remediate

MUST:
- Override the specific managed rule to Count mode: update the web ACL with `RuleActionOverrides` for the offending rule within the managed rule group
- If using a custom rule, narrow the match conditions (add exclusion patterns, scope to specific URI paths)
- Create a higher-priority Allow rule for known-good traffic patterns if appropriate

SHOULD:
- Use scope-down statements to limit managed rule group evaluation to specific paths
- Add label-based exceptions: let the managed rule add a label (Count mode), then add a custom rule to Allow requests with that label AND a trusted characteristic
- Monitor after changes to confirm false positives are resolved without introducing false negatives

MAY:
- Create a custom rule group with refined versions of the overly broad rules
- Use regex pattern sets for precise matching instead of broad string matching

## Common Issues

- symptoms: "Managed rule AWSManagedRulesCommonRuleSet blocks API requests with JSON payloads"
  diagnosis: "CrossSiteScripting_BODY or SQLi_BODY rule matches patterns in JSON field values."
  resolution: "Override the specific rule to Count. Add a scope-down statement to exclude the API path."

- symptoms: "Rate-based rule blocks legitimate high-traffic client"
  diagnosis: "Single client IP exceeds rate threshold due to legitimate usage patterns."
  resolution: "Add the client IP to an IP allow list with higher priority than the rate-based rule."

- symptoms: "SizeRestrictions_BODY rule blocks file upload requests"
  diagnosis: "Request body exceeds the size constraint configured in the managed rule."
  resolution: "Override SizeRestrictions_BODY to Count for the upload path using scope-down statements."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Override managed rule to Count | YELLOW | Rule behavior change; reversible |
| Add scope-down statement | YELLOW | Rule modification; reversible |
| Create Allow rule for known-good traffic | YELLOW | New rule; can be removed |
| Add label-based exceptions | YELLOW | Rule modification; reversible |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- Changes affecting traffic blocking for all users

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-sampled-requests` | MEDIUM | Request headers and IPs |
| `get-web-acl` | LOW | Rule configuration |
| `describe-managed-rule-group` | LOW | Rule group metadata |
| WAF logs | HIGH | Full request headers and client IPs |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting
- NEVER set the web ACL default action to Block without explicit approval

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Rule override to Count | Revert override via `update-web-acl` |
| Scope-down statement | Remove scope-down via `update-web-acl` |
| Allow rule creation | Delete the rule via `update-web-acl` |
| Label-based exception | Remove exception rule via `update-web-acl` |

## Output Format

```yaml
root_cause: "false_positive — <specific_rule_and_cause>"
evidence:
  - type: sampled_request
    content: "<blocked request details>"
  - type: rule_match
    content: "<rule name, priority, and match condition>"
  - type: waf_log
    content: "<terminatingRuleId and labels>"
severity: HIGH
mitigation:
  immediate: "Override specific rule to Count or add Allow exception"
  long_term: "Implement scope-down statements and label-based exception logic"
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

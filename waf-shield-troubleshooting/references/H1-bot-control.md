---
title: "H1 — Bot Control Rule Issues"
description: "Diagnose AWS WAF Bot Control managed rule group problems"
status: active
severity: HIGH
triggers:
  - "Bot Control"
  - "bot detection"
  - "bot blocking"
  - "bot management"
  - "AWSManagedRulesBotControlRuleSet"
owner: devops-agent
objective: "Identify and fix Bot Control rule configuration issues"
context: "AWS WAF Bot Control is a managed rule group that detects and manages bot traffic. It has two inspection levels: Common (included with WAF, detects common bots) and Targeted ($10 per million requests, detects sophisticated bots using behavioral analysis). Bot Control adds labels to requests (awswaf:managed:aws:bot-control:bot:category:<type>) that can be used in custom rules. Common bots include search engines, social media crawlers, and monitoring services."
---

## Phase 1 — Triage

MUST:
- Check if Bot Control is in the web ACL: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id> --query 'WebACL.Rules[?Statement.ManagedRuleGroupStatement.Name==`AWSManagedRulesBotControlRuleSet`]'`
- Check the inspection level (Common or Targeted): look for `ManagedRuleGroupConfigs` in the rule statement
- Verify the override action (None or Count)
- Check individual rule overrides: look for `RuleActionOverrides`
- Get sampled requests for Bot Control: `aws wafv2 get-sampled-requests --web-acl-arn <acl-arn> --rule-metric-name AWS-AWSManagedRulesBotControlRuleSet --scope <scope> --time-window StartTime=<start>,EndTime=<end> --max-items 100`

SHOULD:
- Check labels applied by Bot Control in WAF logs
- Review CloudWatch metrics for Bot Control rule matches
- Verify the scope-down statement if one is configured

MAY:
- Analyze bot traffic patterns from WAF logs
- Check if legitimate bots (Googlebot, etc.) are being blocked

## Phase 2 — Remediate

MUST:
- Start with Bot Control in Count mode to assess impact before blocking
- Override specific bot categories to Allow if they are legitimate (e.g., search engine crawlers)
- Use scope-down statements to apply Bot Control only to specific paths

SHOULD:
- Create custom rules using Bot Control labels for granular control
- Allow verified bots (search engines) while blocking unverified bots
- Monitor Bot Control costs (Targeted level: $10 per million requests)

MAY:
- Implement CAPTCHA or Challenge actions for suspected bots instead of blocking
- Use Bot Control labels with rate-based rules for bot-specific rate limiting
- Create custom response bodies for bot-blocked requests

## Common Issues

- symptoms: "Legitimate search engine crawlers blocked by Bot Control"
  diagnosis: "Bot Control blocks unverified bots. Crawler may not be verified (IP doesn't match known ranges)."
  resolution: "Check if the bot is verified (label: awswaf:managed:aws:bot-control:bot:verified). Override CategorySearchEngine to Allow for verified bots."

- symptoms: "Bot Control not detecting sophisticated bots"
  diagnosis: "Using Common inspection level which only detects common bot signatures."
  resolution: "Upgrade to Targeted inspection level for behavioral analysis ($10/million requests)."

- symptoms: "High Bot Control costs"
  diagnosis: "Targeted inspection level applied to all traffic including static assets."
  resolution: "Add scope-down statement to apply Bot Control only to dynamic paths (API, login)."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Override bot categories to Allow | YELLOW | Rule behavior change; reversible |
| Add scope-down statements | YELLOW | Rule modification; reversible |
| Switch to Count mode | YELLOW | Reduces blocking; reversible |
| Implement CAPTCHA/Challenge | YELLOW | New action type; reversible |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- Bot Control inspection level changes (cost impact)

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl` (Bot Control) | LOW | Rule configuration |
| `get-sampled-requests` | MEDIUM | Request headers and bot labels |
| WAF logs (bot labels) | MEDIUM | Bot classification data |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting
- NEVER disable Bot Control entirely without replacement protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Bot category override | Revert override via `update-web-acl` |
| Scope-down addition | Remove scope-down via `update-web-acl` |
| Count mode switch | Revert to Block via `update-web-acl` |
| CAPTCHA/Challenge action | Revert to previous action via `update-web-acl` |

## Output Format

```yaml
root_cause: "bot_control — <specific_cause>"
evidence:
  - type: inspection_level
    content: "<Common or Targeted>"
  - type: override_action
    content: "<None or Count>"
  - type: bot_labels
    content: "<labels applied to requests>"
severity: HIGH
mitigation:
  immediate: "Adjust Bot Control overrides and scope-down statements"
  long_term: "Implement label-based custom rules for granular bot management"
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

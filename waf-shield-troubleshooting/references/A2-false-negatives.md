---
title: "A2 — WAF False Negatives"
description: "Diagnose why WAF is not blocking malicious traffic that should be caught"
status: active
severity: CRITICAL
triggers:
  - "false negative"
  - "malicious traffic not blocked"
  - "WAF not blocking"
  - "attack getting through"
  - "SQL injection not caught"
owner: devops-agent
objective: "Identify why malicious requests are passing through WAF rules and close the gap"
context: "False negatives occur when malicious requests bypass WAF rules. Common causes include rules set to Count instead of Block, body inspection limits allowing payloads beyond the inspection window, missing rule groups, incorrect rule priority allowing an Allow rule to match before a Block rule, and encoding/obfuscation bypassing pattern matching."
---

## Phase 1 — Triage

MUST:
- Get the web ACL and verify all rules and their actions: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id>`
- Check if the relevant managed rule group is present and not set to Count override: look for `OverrideAction` on the managed rule group statement
- Verify rule priorities — an Allow rule with lower priority number may be matching before the Block rule
- Check body inspection limit: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id> --query 'WebACL.AssociationConfig'`
- Check sampled requests for allowed requests that should have been blocked: `aws wafv2 get-sampled-requests --web-acl-arn <acl-arn> --rule-metric-name <metric-name> --scope <scope> --time-window StartTime=<start>,EndTime=<end> --max-items 100`

SHOULD:
- Verify text transformations are applied (URL decode, HTML entity decode, lowercase) to catch encoded attacks
- Check if the web ACL default action is Allow — unmatched requests pass through
- Review if the attack payload is in the request body beyond the inspection limit (default 8 KB)

MAY:
- Test with known attack patterns to verify rule coverage: `curl -v "https://<domain>/path?id=1' OR '1'='1"`
- Check WAF logs for requests that were allowed but match known attack signatures

## Phase 2 — Remediate

MUST:
- Switch rules from Count to Block if they were in testing mode and are ready for enforcement
- Add missing managed rule groups (AWSManagedRulesCommonRuleSet, AWSManagedRulesSQLiRuleSet, AWSManagedRulesKnownBadInputsRuleSet)
- Fix rule priority ordering so Block rules evaluate before overly broad Allow rules
- Increase body inspection limit if attacks are in the body beyond 8 KB: update to 16/32/64 KB for REGIONAL

SHOULD:
- Add text transformations (URL_DECODE, HTML_ENTITY_DECODE, LOWERCASE) to catch encoded payloads
- Enable WAF logging to capture all requests for analysis
- Add rate-based rules to limit request volume from suspicious sources

MAY:
- Create custom rules for application-specific attack patterns
- Implement label-based multi-stage detection logic

## Common Issues

- symptoms: "SQL injection attacks getting through WAF"
  diagnosis: "AWSManagedRulesSQLiRuleSet is not added to the web ACL, or is set to Count override."
  resolution: "Add the SQL injection managed rule group with OverrideAction: None (use rule group actions)."

- symptoms: "Large POST body attacks bypassing WAF"
  diagnosis: "Attack payload is beyond the 8 KB default body inspection limit."
  resolution: "Increase body inspection limit to 16/32/64 KB for REGIONAL web ACLs."

- symptoms: "Encoded XSS attacks not detected"
  diagnosis: "Rules lack text transformations to decode URL-encoded or HTML-entity-encoded payloads."
  resolution: "Add URL_DECODE and HTML_ENTITY_DECODE text transformations to XSS rules."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Switch rules from Count to Block | YELLOW | May block legitimate traffic; reversible |
| Add managed rule groups | YELLOW | New rules; can be removed |
| Fix rule priority ordering | YELLOW | Evaluation order change; reversible |
| Increase body inspection limit | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- Switching rules from Count to Block in production

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl` | LOW | Rule configuration |
| `get-sampled-requests` | MEDIUM | Request headers and IPs |
| WAF logs | HIGH | Full request headers and client IPs |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting
- NEVER remove security rules without adding replacement coverage

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Rule switch to Block | Revert to Count via `update-web-acl` |
| Managed rule group addition | Remove group via `update-web-acl` |
| Priority reorder | Revert priorities via `update-web-acl` |
| Body inspection limit increase | Revert limit via `update-web-acl` |

## Output Format

```yaml
root_cause: "false_negative — <specific_cause>"
evidence:
  - type: web_acl_config
    content: "<missing or misconfigured rules>"
  - type: sampled_request
    content: "<allowed malicious request details>"
  - type: body_inspection_limit
    content: "<current limit vs payload size>"
severity: CRITICAL
mitigation:
  immediate: "Add missing rules or switch from Count to Block"
  long_term: "Implement comprehensive rule coverage with text transformations and increased body inspection"
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

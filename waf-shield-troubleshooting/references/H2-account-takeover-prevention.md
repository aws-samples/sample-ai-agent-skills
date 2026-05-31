---
title: "H2 — Account Takeover Prevention (ATP) Issues"
description: "Diagnose AWS WAF Account Takeover Prevention managed rule group problems"
status: active
severity: HIGH
triggers:
  - "ATP"
  - "account takeover"
  - "credential stuffing"
  - "login protection"
  - "AWSManagedRulesATPRuleSet"
owner: devops-agent
objective: "Identify and fix Account Takeover Prevention configuration issues"
context: "AWS WAF ATP is a managed rule group that protects login endpoints against credential stuffing and account takeover attacks. It costs $10 per thousand login attempts analyzed. ATP inspects login requests for stolen credentials (checked against a database of compromised credentials), volumetric login anomalies, and distributed login attempts. It requires configuration of the login endpoint path and request body field names for username and password."
---

## Phase 1 — Triage

MUST:
- Check if ATP is in the web ACL: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id> --query 'WebACL.Rules[?Statement.ManagedRuleGroupStatement.Name==`AWSManagedRulesATPRuleSet`]'`
- Verify the login endpoint configuration in ManagedRuleGroupConfigs (LoginPath, RequestInspection)
- Check the override action and individual rule overrides
- Get sampled requests for ATP: `aws wafv2 get-sampled-requests --web-acl-arn <acl-arn> --rule-metric-name AWS-AWSManagedRulesATPRuleSet --scope <scope> --time-window StartTime=<start>,EndTime=<end> --max-items 100`

SHOULD:
- Verify the login path matches the actual application login endpoint
- Check that username and password field names match the request body format
- Review ATP labels in WAF logs for detection accuracy

MAY:
- Analyze login attempt patterns from WAF logs
- Check ATP costs in AWS Cost Explorer
- Test with known compromised credentials (in a test environment)

## Phase 2 — Remediate

MUST:
- Configure the correct login endpoint path (e.g., /api/login, /auth/signin)
- Set the correct request body field names for username and password
- Specify the payload type (JSON or FORM_ENCODED)
- Start in Count mode to validate detection accuracy

SHOULD:
- Configure response inspection to detect successful vs failed logins
- Use ATP labels with custom rules for graduated responses (CAPTCHA for suspicious, Block for confirmed)
- Monitor ATP costs and optimize scope-down to reduce analyzed requests

MAY:
- Implement CAPTCHA challenges for requests labeled as suspicious
- Combine ATP with Bot Control for comprehensive login protection
- Set up CloudWatch alarms for ATP detection spikes

## Common Issues

- symptoms: "ATP not detecting credential stuffing attacks"
  diagnosis: "Login path or request body field names are misconfigured."
  resolution: "Verify LoginPath matches the actual endpoint. Check username/password field names and payload type."

- symptoms: "ATP blocking legitimate login attempts"
  diagnosis: "Legitimate users' credentials appear in the compromised credentials database."
  resolution: "Override CompromisedCredentials rules to Count. Implement password reset flow for affected users."

- symptoms: "ATP costs unexpectedly high"
  diagnosis: "ATP is analyzing all POST requests, not just login attempts."
  resolution: "Verify the login path is specific. Add scope-down statement to limit ATP to the login endpoint only."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Configure login endpoint | YELLOW | Configuration change; reversible |
| Start in Count mode | YELLOW | Non-blocking; reversible |
| Configure response inspection | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- ATP configuration changes affecting login protection

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl` (ATP config) | MEDIUM | Login endpoint path and field names |
| `get-sampled-requests` | HIGH | Login request details |
| WAF logs (ATP labels) | HIGH | Credential compromise indicators |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting
- NEVER expose login credentials in diagnostic output

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Login endpoint config | Revert via `update-web-acl` |
| Count mode switch | Revert to Block via `update-web-acl` |
| Response inspection config | Revert via `update-web-acl` |

## Output Format

```yaml
root_cause: "atp — <specific_cause>"
evidence:
  - type: login_config
    content: "<login path, field names, payload type>"
  - type: override_action
    content: "<None or Count>"
  - type: atp_labels
    content: "<labels applied to login requests>"
severity: HIGH
mitigation:
  immediate: "Fix login endpoint configuration"
  long_term: "Implement graduated response with CAPTCHA and custom rules"
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

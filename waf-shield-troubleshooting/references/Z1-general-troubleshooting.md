---
title: "Z1 — General WAF & Shield Troubleshooting"
description: "Catch-all runbook for WAF and Shield issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "WAF issue"
  - "Shield issue"
  - "WAF not working"
  - "WAF problem"
  - "Shield problem"
owner: devops-agent
objective: "Systematically diagnose any WAF or Shield issue using a general investigation workflow"
context: "This runbook covers general WAF and Shield troubleshooting when the specific issue is unclear. It provides a systematic approach to identify the problem category and route to the appropriate specific runbook. Start with broad data collection and narrow down based on findings."
---

## Phase 1 — Triage

MUST:
- Get the web ACL configuration: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id>`
- Check web ACL association: `aws wafv2 list-resources-for-web-acl --web-acl-arn <web-acl-arn>`
- Check CloudWatch metrics for blocked and allowed requests: `aws cloudwatch get-metric-statistics --namespace AWS/WAFV2 --metric-name BlockedRequests --dimensions Name=WebACL,Value=<acl-name> Name=Rule,Value=ALL Name=Region,Value=<region> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check sampled requests for recent activity: `aws wafv2 get-sampled-requests --web-acl-arn <acl-arn> --rule-metric-name <metric-name> --scope <scope> --time-window StartTime=<start>,EndTime=<end> --max-items 100`
- Check Shield subscription status: `aws shield describe-subscription`

SHOULD:
- Check WAF logging configuration: `aws wafv2 get-logging-configuration --resource-arn <web-acl-arn>`
- List all IP sets and regex pattern sets: `aws wafv2 list-ip-sets --scope <scope>` and `aws wafv2 list-regex-pattern-sets --scope <scope>`
- Check CloudTrail for recent WAF/Shield API calls: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=wafv2.amazonaws.com --start-time <start> --end-time <end>`
- Check Shield protections: `aws shield list-protections`

MAY:
- Check AWS Health Dashboard for WAF/Shield service issues
- Review recent deployments or configuration changes
- Check AWS Firewall Manager policies if in use

## Phase 2 — Remediate

MUST:
- Route to the appropriate specific runbook based on findings:
  - False positives → A1
  - False negatives → A2
  - Rule priority issues → A3
  - Rate-based rule issues → A4
  - Managed rule issues → B1-B3
  - IP/Geo issues → C1-C3
  - Body/regex/size issues → D1-D3
  - Logging/monitoring issues → E1-E3
  - Integration issues → F1-F3
  - DDoS/Shield issues → G1-G4
  - Bot Control issues → H1-H2
  - Custom response issues → I1-I2

SHOULD:
- Document the investigation findings and resolution
- Update monitoring and alerting based on the issue discovered
- Review the guardrails in waf-shield-guardrails.md

MAY:
- Conduct a broader WAF configuration review
- Implement preventive measures for the identified issue category

## Common Issues

- symptoms: "WAF appears to have no effect on traffic"
  diagnosis: "Web ACL is not associated with any resource."
  resolution: "Associate the web ACL with the target resource (ALB, CloudFront, API Gateway)."

- symptoms: "Intermittent WAF behavior — sometimes blocks, sometimes allows"
  diagnosis: "Multiple rules with overlapping conditions and different actions. Priority ordering causes inconsistent behavior depending on request characteristics."
  resolution: "Review rule priorities and match conditions. Simplify overlapping rules."

- symptoms: "WAF changes not taking effect"
  diagnosis: "Web ACL update is still propagating. Changes can take a few seconds to minutes."
  resolution: "Wait for propagation. Verify the update was successful via get-web-acl."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Route to specific runbook | GREEN | Classification only; no state change |
| Investigate and collect evidence | GREEN | Read-only analysis |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- Unclassified issues after investigation
- Any issue affecting traffic for all users

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl` | LOW | Rule configuration |
| `list-resources-for-web-acl` | LOW | Associated resource ARNs |
| `get-sampled-requests` | MEDIUM | Request headers and IPs |
| CloudWatch metrics | LOW | Aggregate request counts |
| CloudTrail events | MEDIUM | API call history |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Route to specific runbook | Follow that runbook's rollback procedures |
| No direct remediation in catch-all | N/A — remediation happens in specific runbooks |

## Output Format

```yaml
root_cause: "general — <identified_category_and_cause>"
evidence:
  - type: web_acl_config
    content: "<web ACL summary>"
  - type: association
    content: "<associated resources>"
  - type: metrics
    content: "<CloudWatch metric summary>"
severity: MEDIUM
mitigation:
  immediate: "Follow the specific runbook for the identified issue category"
  long_term: "Implement comprehensive WAF monitoring and configuration management"
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

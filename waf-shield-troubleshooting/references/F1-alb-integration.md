---
title: "F1 — ALB + WAF Integration Issues"
description: "Diagnose WAF integration problems with Application Load Balancers"
status: active
severity: HIGH
triggers:
  - "ALB WAF"
  - "load balancer WAF"
  - "ALB integration"
  - "WAF not protecting ALB"
  - "ALB 403"
owner: devops-agent
objective: "Identify and fix WAF integration issues with Application Load Balancers"
context: "WAF can be associated with ALBs using REGIONAL scope web ACLs. The web ACL must be in the same region as the ALB. WAF evaluates requests after the ALB receives them but before routing to targets. Common issues include scope mismatch, region mismatch, missing association, and X-Forwarded-For header handling for IP-based rules."
---

## Phase 1 — Triage

MUST:
- Check if a web ACL is associated with the ALB: `aws wafv2 get-web-acl-for-resource --resource-arn <alb-arn>`
- Verify the web ACL scope is REGIONAL (not CLOUDFRONT): `aws wafv2 get-web-acl --name <acl-name> --scope REGIONAL --id <acl-id>`
- Verify the web ACL and ALB are in the same region
- Check CloudWatch WAF metrics for the web ACL to confirm traffic is being evaluated

SHOULD:
- Verify the ALB security group allows inbound traffic (WAF cannot block if traffic doesn't reach the ALB)
- Check if the ALB has multiple web ACLs (only one can be associated)
- Review ALB access logs for correlation with WAF blocks

MAY:
- Check CloudTrail for AssociateWebACL events
- Verify the ALB is in an active state

## Phase 2 — Remediate

MUST:
- Associate the web ACL with the ALB: `aws wafv2 associate-web-acl --web-acl-arn <web-acl-arn> --resource-arn <alb-arn>`
- Ensure the web ACL is REGIONAL scope and in the same region as the ALB
- Configure IP-based rules to use X-Forwarded-For if clients are behind a proxy

SHOULD:
- Enable WAF logging to monitor requests hitting the ALB
- Set up CloudWatch alarms for WAF metrics on the ALB
- Test the association by sending a request that should be blocked

MAY:
- Use AWS Firewall Manager to manage WAF associations across multiple ALBs
- Implement WAF association as part of ALB deployment automation

## Common Issues

- symptoms: "WAF not blocking any requests to the ALB"
  diagnosis: "Web ACL is not associated with the ALB, or all rules are in Count mode."
  resolution: "Associate the web ACL. Verify rules are not all overridden to Count."

- symptoms: "AssociateWebACL fails with WAFInvalidParameterException"
  diagnosis: "Web ACL scope is CLOUDFRONT (must be REGIONAL for ALB) or region mismatch."
  resolution: "Create a REGIONAL web ACL in the same region as the ALB."

- symptoms: "IP-based rules not matching correct client IPs behind ALB"
  diagnosis: "WAF sees the last-hop IP. For clients behind proxies, use ForwardedIPConfig."
  resolution: "Configure IP rules with ForwardedIPConfig and X-Forwarded-For header."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Associate web ACL with ALB | YELLOW | Enables WAF evaluation; reversible |
| Configure ForwardedIPConfig | YELLOW | Rule modification; reversible |
| Enable WAF logging | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- Web ACL association changes for production ALBs

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl-for-resource` | LOW | Association status |
| `get-web-acl` | LOW | Rule configuration |
| CloudWatch WAF metrics | LOW | Aggregate request counts |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting
- NEVER disassociate WAF from a production ALB without explicit approval

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Web ACL association | Disassociate via `disassociate-web-acl` |
| ForwardedIPConfig change | Revert via `update-web-acl` |

## Output Format

```yaml
root_cause: "alb_integration — <specific_cause>"
evidence:
  - type: association
    content: "<web ACL association status>"
  - type: scope
    content: "<web ACL scope and region>"
  - type: alb_status
    content: "<ALB state and configuration>"
severity: HIGH
mitigation:
  immediate: "Associate web ACL or fix scope/region mismatch"
  long_term: "Automate WAF association in ALB deployment pipeline"
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

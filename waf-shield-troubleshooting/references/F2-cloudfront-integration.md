---
title: "F2 — CloudFront + WAF Integration Issues"
description: "Diagnose WAF integration problems with CloudFront distributions"
status: active
severity: HIGH
triggers:
  - "CloudFront WAF"
  - "distribution WAF"
  - "CloudFront integration"
  - "WAF not protecting CloudFront"
owner: devops-agent
objective: "Identify and fix WAF integration issues with CloudFront distributions"
context: "WAF for CloudFront requires CLOUDFRONT scope web ACLs created in us-east-1. The web ACL is associated at the distribution level and applies to all cache behaviors. WAF evaluates requests at CloudFront edge locations before caching. Body inspection is limited to 8 KB for CloudFront (cannot be increased). IP-based rules see the viewer's IP directly (no X-Forwarded-For needed)."
---

## Phase 1 — Triage

MUST:
- Check if a web ACL is associated with the distribution: `aws cloudfront get-distribution --id <dist-id> --query 'Distribution.DistributionConfig.WebACLId'`
- Verify the web ACL scope is CLOUDFRONT and created in us-east-1: `aws wafv2 get-web-acl --name <acl-name> --scope CLOUDFRONT --id <acl-id> --region us-east-1`
- Check CloudWatch WAF metrics (use Region=us-east-1 for CLOUDFRONT scope)
- Verify the distribution status is Deployed

SHOULD:
- Check if the web ACL has rules appropriate for edge evaluation
- Verify body inspection limit awareness (fixed 8 KB for CloudFront)
- Review CloudFront access logs for correlation with WAF actions

MAY:
- Check CloudTrail for UpdateDistribution events that may have changed the web ACL association
- Verify the web ACL ARN in the distribution config

## Phase 2 — Remediate

MUST:
- Create the web ACL in us-east-1 with CLOUDFRONT scope
- Associate via CloudFront distribution config: `aws cloudfront update-distribution --id <dist-id> --distribution-config <config-with-web-acl-id> --if-match <etag>`
- Or associate via WAF: `aws wafv2 associate-web-acl --web-acl-arn <web-acl-arn> --resource-arn <distribution-arn> --region us-east-1`

SHOULD:
- Account for the 8 KB body inspection limit in rule design
- Use CloudFront-specific managed rule groups where available
- Test WAF rules with edge-location traffic patterns

MAY:
- Use AWS Firewall Manager for cross-distribution WAF management
- Combine WAF with CloudFront geographic restrictions for layered protection

## Common Issues

- symptoms: "Cannot associate web ACL with CloudFront distribution"
  diagnosis: "Web ACL is REGIONAL scope or not in us-east-1."
  resolution: "Create a CLOUDFRONT scope web ACL in us-east-1."

- symptoms: "WAF rules not blocking at CloudFront edge"
  diagnosis: "Web ACL is associated but all rules are in Count mode, or the distribution is still deploying."
  resolution: "Check rule actions. Wait for distribution deployment to complete (5-10 minutes)."

- symptoms: "Body-based rules not catching attacks through CloudFront"
  diagnosis: "Body inspection limit is fixed at 8 KB for CloudFront and cannot be increased."
  resolution: "Implement application-level body validation. Use origin WAF (REGIONAL on ALB) for larger body inspection."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Associate web ACL with CloudFront | YELLOW | Enables WAF; reversible |
| Create CLOUDFRONT scope web ACL | YELLOW | New resource; can be deleted |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- CloudFront distribution configuration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-distribution` (WebACLId) | LOW | Association status |
| `get-web-acl` | LOW | Rule configuration |
| CloudWatch WAF metrics | LOW | Aggregate request counts |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting
- NEVER disassociate WAF from a production CloudFront distribution without explicit approval

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Web ACL association | Remove WebACLId from distribution config |
| New web ACL creation | Delete web ACL via `delete-web-acl` |

## Output Format

```yaml
root_cause: "cloudfront_integration — <specific_cause>"
evidence:
  - type: association
    content: "<web ACL ID in distribution config>"
  - type: scope
    content: "<web ACL scope and region>"
  - type: distribution_status
    content: "<Deployed or InProgress>"
severity: HIGH
mitigation:
  immediate: "Fix scope/region or associate web ACL"
  long_term: "Design rules accounting for CloudFront-specific limitations"
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

---
title: "B2 — Marketplace Rule Group Issues"
description: "Diagnose problems with AWS Marketplace WAF rule groups"
status: active
severity: MEDIUM
triggers:
  - "marketplace rules"
  - "third-party rules"
  - "vendor rule group"
  - "marketplace WAF"
owner: devops-agent
objective: "Identify and resolve issues with third-party marketplace WAF rule groups"
context: "AWS Marketplace WAF rule groups are created by third-party security vendors. They function like AWS managed rule groups but are subscribed to through AWS Marketplace. They have their own WCU costs, versioning, and update schedules controlled by the vendor. Troubleshooting requires understanding both WAF mechanics and vendor-specific rule behavior."
---

## Phase 1 — Triage

MUST:
- Identify marketplace rule groups in the web ACL: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id> --query 'WebACL.Rules[?Statement.ManagedRuleGroupStatement]'`
- Check the vendor name and rule group name for marketplace groups (vendor name is not "AWS")
- Describe the marketplace rule group: `aws wafv2 describe-managed-rule-group --vendor-name <vendor-name> --name <rule-group-name> --scope <scope>`
- Verify the Marketplace subscription is active
- Check sampled requests for blocks by the marketplace rule group

SHOULD:
- Check the WCU consumption of the marketplace rule group
- Verify the override action and any individual rule overrides
- Review CloudWatch metrics for the marketplace rule group

MAY:
- Contact the vendor for rule-specific documentation
- Check if the vendor has published release notes for recent version changes

## Phase 2 — Remediate

MUST:
- Apply the same override and scope-down techniques as AWS managed rules
- Verify the Marketplace subscription has not expired
- Ensure the rule group scope matches the web ACL scope (REGIONAL or CLOUDFRONT)

SHOULD:
- Test marketplace rule group updates in Count mode before enforcing
- Monitor WCU consumption as vendor updates may change WCU requirements
- Document vendor contact information for escalation

MAY:
- Evaluate alternative marketplace or AWS managed rule groups if issues persist
- Request custom rule tuning from the vendor

## Common Issues

- symptoms: "Marketplace rule group disappeared from web ACL"
  diagnosis: "Marketplace subscription expired or was cancelled."
  resolution: "Renew the Marketplace subscription. Re-add the rule group to the web ACL."

- symptoms: "Marketplace rule group blocking with no visibility into which rule"
  diagnosis: "Vendor rule groups may not expose individual rule names in sampled requests."
  resolution: "Check WAF logs for labels. Contact vendor for rule documentation."

- symptoms: "WCU limit exceeded after marketplace rule group update"
  diagnosis: "Vendor updated the rule group, increasing its WCU consumption."
  resolution: "Pin to previous version or remove other rules to free WCU capacity."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Apply override/scope-down techniques | YELLOW | Rule modification; reversible |
| Renew Marketplace subscription | YELLOW | Billing change; recoverable |
| Evaluate alternative rule groups | GREEN | Assessment only; no state change |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- Marketplace subscription changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl` (marketplace rules) | LOW | Rule configuration |
| `describe-managed-rule-group` | LOW | Rule group metadata |
| `get-sampled-requests` | MEDIUM | Request headers and IPs |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Override/scope-down change | Revert via `update-web-acl` |
| Subscription renewal | Cancel via AWS Marketplace |

## Output Format

```yaml
root_cause: "marketplace_rules — <specific_cause>"
evidence:
  - type: vendor_info
    content: "<vendor name and rule group>"
  - type: subscription_status
    content: "<active or expired>"
  - type: wcu_consumption
    content: "<WCU used by the group>"
severity: MEDIUM
mitigation:
  immediate: "Fix override action or renew subscription"
  long_term: "Implement version pinning and vendor communication process"
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

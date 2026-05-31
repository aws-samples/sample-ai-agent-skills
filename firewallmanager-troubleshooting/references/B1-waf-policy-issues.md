---
title: "B1 â€” WAF Policy Issues"
description: "Diagnose WAF policy problems in Firewall Manager"
status: active
severity: HIGH
triggers:
  - "WAF policy not working"
  - "web ACL not created"
  - "WAF rules not applied"
  - "FMS WAF policy error"
owner: devops-agent
objective: "Identify and resolve WAF policy issues in Firewall Manager"
context: "FMS WAF policies create and manage WAF web ACLs in member accounts. Issues can arise from incorrect rule group configuration, resource type mismatches, web ACL association failures, or conflicts with existing web ACLs. FMS-managed web ACLs use the FMManagedWebACLV2- prefix."
---

## Phase 1 â€” Triage

MUST:
- Get the WAF policy details: `aws fms get-policy --policy-id <policy-id>`
- Check compliance status: `aws fms list-compliance-status --policy-id <policy-id>`
- List WAF web ACLs in member account: `aws wafv2 list-web-acls --scope REGIONAL`
- Check if FMS-managed web ACL exists: look for FMManagedWebACLV2- prefix

SHOULD:
- Verify rule groups referenced in the policy exist: `aws wafv2 list-rule-groups --scope REGIONAL`
- Check web ACL associations: `aws wafv2 list-resources-for-web-acl --web-acl-arn <arn>`
- Verify the resource type in policy scope matches target resources
- Check for existing web ACLs that may conflict

MAY:
- Review WAF logging for rule evaluation: `aws wafv2 get-logging-configuration --resource-arn <web-acl-arn>`
- Check CloudWatch metrics for WAF rule matches

## Phase 2 â€” Remediate

MUST:
- Ensure rule groups referenced in the policy are valid and accessible
- Verify policy scope includes the correct resource types
- Check that resources don't already have a non-FMS web ACL associated

SHOULD:
- Use first/last rule groups in FMS policy for custom rules
- Verify web ACL capacity units (WCU) are within limits
- Test policy with a small scope before expanding

MAY:
- Enable WAF logging for troubleshooting
- Set up CloudWatch dashboards for WAF metrics

## Common Issues

- symptoms: "Web ACL created but not associated with resources"
  diagnosis: "Resource type mismatch or resource already has a web ACL."
  resolution: "Verify resource type in policy. Remove existing web ACL associations."

- symptoms: "Rule group not found error"
  diagnosis: "Rule group ARN is invalid or not accessible from member account."
  resolution: "Verify rule group exists and is shared with member accounts if needed."

## Output Format

```yaml
root_cause: "waf_policy_issue â€” <specific_cause>"
evidence:
  - type: policy_config
    content: "<WAF policy configuration>"
  - type: web_acl_status
    content: "<web ACL creation and association status>"
severity: HIGH
mitigation:
  immediate: "Fix the specific WAF policy issue"
  long_term: "Standardize WAF rule groups and automate policy validation"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | CRITICAL — Firewall Manager enforces security policies across the entire organization |
| Data Risk | HIGH — Policy changes can expose resources or block legitimate traffic |
| Recovery Complexity | HIGH — Organization-wide policy rollbacks require coordinated multi-account changes |
| Blast Radius | CRITICAL — Policy changes propagate to all member accounts in scope |

## Escalation Conditions

- Security policy non-compliance detected across multiple member accounts
- Auto-remediation actions causing unintended resource modifications
- WAF rule group changes blocking legitimate production traffic
- Shield Advanced policy gaps leaving resources unprotected during active DDoS
- Network Firewall policy deployment failures leaving VPCs without inspection
- DNS Firewall policy blocking legitimate domain resolutions

## Data Sensitivity

- Security policies reveal organizational security posture and architecture
- Member account compliance data exposes security gaps across the organization
- WAF rule configurations may contain IP allowlists and rate limiting thresholds
- Admin account credentials grant organization-wide security policy control
- Audit policy results reveal security group configurations across all accounts

## PROHIBITED Actions

- NEVER suggest removing the Firewall Manager administrator account without a replacement
- NEVER disable auto-remediation on policies protecting against active threats
- NEVER modify WAF rule groups without testing impact on legitimate traffic patterns
- NEVER delete Shield Advanced policies during an active DDoS event
- NEVER change security group audit policies without notifying affected account owners
- NEVER deploy Network Firewall policy changes to all accounts simultaneously without staged rollout

## Phase 3 — Rollback

- If policy change blocked legitimate traffic: revert policy configuration and verify traffic flow restoration
- If auto-remediation caused issues: disable auto-remediation, manually restore affected resources, then fix policy
- If WAF rule group update broke applications: revert to previous rule group version and clear WAF logs
- If security group policy caused compliance failures: restore previous policy scope and re-evaluate compliance
- If Network Firewall deployment failed: revert to previous firewall policy and verify VPC traffic inspection
- Document all rollback actions with policy ID, affected accounts, and compliance status before/after
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"

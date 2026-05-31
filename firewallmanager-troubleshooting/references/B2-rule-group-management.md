---
title: "B2 â€” Rule Group Management"
description: "Diagnose WAF rule group issues in Firewall Manager policies"
status: active
severity: MEDIUM
triggers:
  - "rule group error"
  - "rule group not applied"
  - "WCU limit exceeded"
  - "rule group conflict"
owner: devops-agent
objective: "Identify and resolve WAF rule group management issues in FMS"
context: "FMS WAF policies reference rule groups as first rule groups (evaluated before custom rules) and last rule groups (evaluated after). Rule groups have Web ACL Capacity Unit (WCU) limits. Managed rule groups from AWS or Marketplace can be used. Rule group sharing across accounts requires proper configuration."
---

## Phase 1 â€” Triage

MUST:
- List rule groups in the account: `aws wafv2 list-rule-groups --scope REGIONAL`
- Get rule group details: `aws wafv2 get-rule-group --scope REGIONAL --name <name> --id <id>`
- Check WCU usage: `aws wafv2 get-web-acl --scope REGIONAL --name <name> --id <id> --query 'WebACL.Capacity'`
- Verify FMS policy rule group references: `aws fms get-policy --policy-id <policy-id>`

SHOULD:
- List available managed rule groups: `aws wafv2 list-available-managed-rule-groups --scope REGIONAL`
- Check rule group capacity against web ACL WCU limit (5000 WCU default)
- Verify rule group is accessible from member accounts
- Check for rule group version updates

MAY:
- Review rule group rules for conflicts
- Check CloudWatch metrics for rule group evaluation

## Phase 2 â€” Remediate

MUST:
- Ensure total WCU of all rule groups does not exceed web ACL limit
- Verify rule group ARNs in FMS policy are correct
- For managed rule groups, ensure subscription is active

SHOULD:
- Use first/last rule group ordering appropriately in FMS policy
- Test rule group changes in a staging environment
- Monitor rule group evaluation metrics after changes

MAY:
- Create custom rule groups for organization-specific rules
- Set up automated rule group updates

## Common Issues

- symptoms: "WCU limit exceeded when adding rule group"
  diagnosis: "Total WCU of all rule groups exceeds the 5000 WCU web ACL limit."
  resolution: "Remove or optimize rule groups to stay within WCU limits. Request limit increase if needed."

- symptoms: "Managed rule group not available"
  diagnosis: "Managed rule group subscription expired or not available in region."
  resolution: "Verify subscription status. Check regional availability of the managed rule group."

## Output Format

```yaml
root_cause: "rule_group_issue â€” <specific_cause>"
evidence:
  - type: rule_groups
    content: "<rule group configuration and WCU>"
  - type: policy_config
    content: "<FMS policy rule group references>"
severity: MEDIUM
mitigation:
  immediate: "Fix the specific rule group issue"
  long_term: "Implement WCU budgeting and rule group lifecycle management"
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

---
title: "F2 â€” Member Account Compliance"
description: "Diagnose member account compliance issues with FMS policies"
status: active
severity: HIGH
triggers:
  - "member account non-compliant"
  - "account not in scope"
  - "policy not applied to account"
  - "member compliance"
owner: devops-agent
objective: "Identify and resolve member account compliance issues with FMS policies"
context: "FMS policies apply to member accounts in the organization based on scope configuration. Member accounts can be non-compliant due to scope exclusions, OU membership, missing resources, permission issues, or propagation delays. Compliance is evaluated per account per policy."
---

## Phase 1 â€” Triage

MUST:
- List member accounts: `aws fms list-member-accounts`
- Check compliance for specific account: `aws fms get-compliance-detail --policy-id <policy-id> --member-account <account-id>`
- Verify account is in policy scope: `aws fms get-policy --policy-id <policy-id> --query 'Policy.{IncludeMap:IncludeMap,ExcludeMap:ExcludeMap}'`
- Check account OU membership: `aws organizations list-parents --child-id <account-id>`

SHOULD:
- Verify the account is active in the organization
- Check if the account was recently added (propagation delay)
- Review violation details for non-compliant resources
- Check FMS service-linked role in the member account

MAY:
- Check SCPs that might affect FMS operations in member accounts
- Review CloudTrail in member account for FMS-related events

## Phase 2 â€” Remediate

MUST:
- Ensure the account is included in policy scope (not excluded)
- Verify the account's OU is in the policy scope
- Address specific violations in the member account

SHOULD:
- Wait for propagation if account was recently added
- Verify FMS SLR exists in the member account
- Check for SCPs blocking FMS remediation actions

MAY:
- Set up per-account compliance dashboards
- Create automated compliance reporting

## Common Issues

- symptoms: "Account not showing in compliance status"
  diagnosis: "Account is excluded from policy scope or not in a targeted OU."
  resolution: "Check policy IncludeMap/ExcludeMap. Verify account OU membership."

- symptoms: "Account compliant but resources unprotected"
  diagnosis: "Resources don't match policy resource type or tag scope."
  resolution: "Verify resource tags match policy scope. Check resource type configuration."

## Output Format

```yaml
root_cause: "member_compliance â€” <specific_cause>"
evidence:
  - type: account_scope
    content: "<account inclusion/exclusion in policy>"
  - type: compliance_detail
    content: "<compliance details for the account>"
severity: HIGH
mitigation:
  immediate: "Fix account scope or address specific violations"
  long_term: "Automate compliance monitoring and reporting across accounts"
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

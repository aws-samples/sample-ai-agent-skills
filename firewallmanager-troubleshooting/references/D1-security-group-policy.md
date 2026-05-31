---
title: "D1 â€” Security Group Policy Issues"
description: "Diagnose FMS security group usage policy problems"
status: active
severity: HIGH
triggers:
  - "security group policy not working"
  - "SG rules not applied"
  - "security group non-compliant"
  - "FMS security group"
owner: devops-agent
objective: "Identify and resolve FMS security group usage policy issues"
context: "FMS security group USAGE policies enforce baseline security group rules on resources. They can create replica security groups in member accounts and associate them with in-scope resources. Auto-remediation can add required rules or associate the baseline security group. This is different from AUDIT policies which only report findings."
---

## Phase 1 â€” Triage

MUST:
- Get security group policy details: `aws fms get-policy --policy-id <policy-id>`
- Check compliance status: `aws fms list-compliance-status --policy-id <policy-id>`
- Verify the primary security group referenced in the policy: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check if replica security groups exist in member accounts

SHOULD:
- Verify policy type is SECURITY_GROUPS_USAGE (not AUDIT)
- Check resource scope and tags
- Verify auto-remediation settings
- Review violation details for non-compliant resources

MAY:
- Check CloudTrail for security group modification events
- Review security group rules in member accounts

## Phase 2 â€” Remediate

MUST:
- Ensure the primary security group in the admin account is correctly configured
- Verify policy scope matches target resources
- Enable auto-remediation if appropriate

SHOULD:
- Test policy with a small scope before expanding
- Monitor replica security group creation in member accounts
- Verify security group associations after remediation

MAY:
- Set up compliance notifications
- Create automated compliance reporting

## Common Issues

- symptoms: "Replica security group not created in member account"
  diagnosis: "FMS service-linked role lacks permissions or policy scope doesn't match."
  resolution: "Verify FMS SLR permissions. Check policy scope configuration."

- symptoms: "Security group rules not matching policy"
  diagnosis: "Auto-remediation may not handle all rule conflicts."
  resolution: "Check for conflicting rules. Manually resolve complex conflicts."

## Output Format

```yaml
root_cause: "security_group_policy â€” <specific_cause>"
evidence:
  - type: policy_config
    content: "<security group policy configuration>"
  - type: compliance_status
    content: "<compliance status across accounts>"
severity: HIGH
mitigation:
  immediate: "Fix security group policy configuration"
  long_term: "Standardize security group baselines and automate compliance"
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

---
title: "D2 â€” Security Group Audit Policy Issues"
description: "Diagnose FMS security group audit policy problems"
status: active
severity: MEDIUM
triggers:
  - "audit policy not reporting"
  - "security group audit"
  - "overly permissive security groups"
  - "unused security groups"
owner: devops-agent
objective: "Identify and resolve FMS security group audit policy issues"
context: "FMS security group AUDIT policies report non-compliant security groups without modifying them. They can detect overly permissive rules, unused security groups, and redundant security groups. Audit policies generate findings but do not auto-remediate. This is different from USAGE policies which enforce rules."
---

## Phase 1 â€” Triage

MUST:
- Get audit policy details: `aws fms get-policy --policy-id <policy-id>`
- Verify policy type is SECURITY_GROUPS_AUDIT
- Check compliance status: `aws fms list-compliance-status --policy-id <policy-id>`
- Get violation details: `aws fms get-violation-details --policy-id <policy-id> --member-account <account-id> --resource-id <sg-id> --resource-type AWS::EC2::SecurityGroup`

SHOULD:
- Review audit policy rules (overly permissive, unused, redundant)
- Check policy scope and resource tags
- Verify member accounts are in scope
- Review the audit configuration settings

MAY:
- Export compliance findings for reporting
- Compare findings across accounts

## Phase 2 â€” Remediate

MUST:
- Review audit findings and prioritize by severity
- Manually remediate non-compliant security groups (audit policies don't auto-remediate)
- Document remediation actions taken

SHOULD:
- Create remediation workflows for common findings
- Track remediation progress across accounts
- Update security group baselines based on findings

MAY:
- Automate remediation with Lambda functions triggered by findings
- Create compliance dashboards

## Common Issues

- symptoms: "Audit policy not generating findings"
  diagnosis: "Policy scope doesn't match any resources or evaluation hasn't completed."
  resolution: "Verify policy scope. Wait for evaluation cycle to complete."

- symptoms: "Too many false positive findings"
  diagnosis: "Audit rules are too strict for the environment."
  resolution: "Adjust audit policy rules. Add exceptions for known-good configurations."

## Output Format

```yaml
root_cause: "audit_policy â€” <specific_cause>"
evidence:
  - type: policy_config
    content: "<audit policy configuration>"
  - type: findings
    content: "<audit findings summary>"
severity: MEDIUM
mitigation:
  immediate: "Review and prioritize audit findings"
  long_term: "Establish security group governance and automated remediation"
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

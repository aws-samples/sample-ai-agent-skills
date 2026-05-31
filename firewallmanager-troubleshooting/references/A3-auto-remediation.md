---
title: "A3 â€” Auto-Remediation Issues"
description: "Diagnose why FMS auto-remediation is not working as expected"
status: active
severity: HIGH
triggers:
  - "auto-remediation not working"
  - "remediation failed"
  - "resources not auto-fixed"
  - "RemediationEnabled but still non-compliant"
owner: devops-agent
objective: "Identify why auto-remediation is failing and resolve the issue"
context: "Auto-remediation creates FMS-managed resources in member accounts but has limitations. WAF remediation creates web ACLs, security group remediation modifies rules, Network Firewall remediation deploys endpoints. Some violations require manual intervention. Remediation depends on the FMS service-linked role having proper permissions in member accounts."
---

## Phase 1 â€” Triage

MUST:
- Verify auto-remediation is enabled: `aws fms get-policy --policy-id <policy-id> --query 'Policy.RemediationEnabled'`
- Check compliance detail for remediation status: `aws fms get-compliance-detail --policy-id <policy-id> --member-account <account-id>`
- Verify FMS service-linked role exists in member account
- Check CloudTrail in member account for FMS remediation actions

SHOULD:
- Verify the violation type supports auto-remediation
- Check if there are conflicting resources blocking remediation
- Review the policy configuration for remediation settings
- Check for IAM permission issues in member accounts

MAY:
- Test remediation on a single resource before applying broadly
- Check AWS Health Dashboard for FMS service issues

## Phase 2 â€” Remediate

MUST:
- Ensure RemediationEnabled is true in the policy
- Verify FMS service-linked role has required permissions
- For unsupported violation types, perform manual remediation
- Wait for remediation to complete (can take several minutes)

SHOULD:
- Monitor remediation progress through compliance status
- Address any conflicting resources that block remediation
- Document which violation types require manual intervention

MAY:
- Create Lambda-based custom remediation for unsupported types
- Set up CloudWatch alarms for remediation failures

## Common Issues

- symptoms: "Remediation enabled but web ACL not associated"
  diagnosis: "Resource type may not support automatic web ACL association."
  resolution: "Manually associate the web ACL or verify resource type is supported."

- symptoms: "Security group rules not being applied"
  diagnosis: "Conflicting rules or dependencies preventing remediation."
  resolution: "Check for rule conflicts. Remove conflicting rules manually."

## Output Format

```yaml
root_cause: "auto_remediation_failure â€” <specific_cause>"
evidence:
  - type: remediation_status
    content: "<remediation enabled and status>"
  - type: violation_type
    content: "<violation type and remediation support>"
severity: HIGH
mitigation:
  immediate: "Fix the specific remediation blocker"
  long_term: "Document remediation limitations and create manual runbooks for unsupported types"
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

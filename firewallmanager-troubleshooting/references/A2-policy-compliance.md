---
title: "A2 â€” Policy Compliance Issues"
description: "Diagnose why resources are not compliant with FMS policies"
status: active
severity: HIGH
triggers:
  - "non-compliant resources"
  - "policy compliance failed"
  - "resources not protected"
  - "compliance status NON_COMPLIANT"
owner: devops-agent
objective: "Identify why resources are non-compliant with FMS policies and resolve violations"
context: "Resources can be non-compliant due to missing WAF web ACLs, incorrect security group rules, missing Shield protection, undeployed Network Firewall endpoints, scope misconfiguration, or propagation delays. Compliance evaluation timing varies by policy type."
---

## Phase 1 â€” Triage

MUST:
- Check compliance status for the policy: `aws fms list-compliance-status --policy-id <policy-id>`
- Get compliance detail for a specific account: `aws fms get-compliance-detail --policy-id <policy-id> --member-account <account-id>`
- Get violation details: `aws fms get-violation-details --policy-id <policy-id> --member-account <account-id> --resource-id <resource-id> --resource-type <type>`
- Verify policy scope matches the non-compliant resources: `aws fms get-policy --policy-id <policy-id>`

SHOULD:
- Check if auto-remediation is enabled: `aws fms get-policy --policy-id <policy-id> --query 'Policy.RemediationEnabled'`
- Verify the resource tags match the policy scope
- Check if the resource was recently created (propagation delay)
- Review the specific violation type and required remediation

MAY:
- Check CloudTrail for recent policy or resource changes
- Verify the member account is part of the policy scope

## Phase 2 â€” Remediate

MUST:
- Address the specific violation identified in the compliance detail
- For WAF violations: ensure web ACL is associated with the resource
- For security group violations: update security group rules to match policy
- For Shield violations: enable Shield Advanced protection on the resource

SHOULD:
- Enable auto-remediation if appropriate: update policy with RemediationEnabled=true
- Wait for compliance re-evaluation after remediation
- Verify compliance status after remediation

MAY:
- Set up SNS notifications for compliance changes
- Create automated remediation workflows for recurring violations

## Common Issues

- symptoms: "Resources show NON_COMPLIANT but policy is correct"
  diagnosis: "Propagation delay or resource tags don't match policy scope."
  resolution: "Wait for propagation (up to 1 hour). Verify resource tags match policy scope."

- symptoms: "Auto-remediation enabled but resources still non-compliant"
  diagnosis: "Auto-remediation may not handle all violation types."
  resolution: "Check violation details. Some violations require manual remediation."

## Output Format

```yaml
root_cause: "policy_compliance â€” <specific_violation>"
evidence:
  - type: compliance_status
    content: "<compliance status details>"
  - type: violation_details
    content: "<specific violation information>"
severity: HIGH
mitigation:
  immediate: "Remediate the specific violation"
  long_term: "Enable auto-remediation and set up compliance monitoring"
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

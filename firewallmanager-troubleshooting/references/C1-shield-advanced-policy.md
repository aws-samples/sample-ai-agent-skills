---
title: "C1 â€” Shield Advanced Policy Issues"
description: "Diagnose Shield Advanced policy problems in Firewall Manager"
status: active
severity: HIGH
triggers:
  - "Shield Advanced policy error"
  - "Shield not enabled"
  - "Shield policy non-compliant"
  - "DDoS protection policy"
owner: devops-agent
objective: "Identify and resolve Shield Advanced policy issues in FMS"
context: "FMS Shield Advanced policies enable Shield Advanced protection on resources in member accounts. Shield Advanced requires an active subscription ($3,000/month per organization). Policies can protect CloudFront, ALB, EIP, Global Accelerator, and Route 53 resources. Shield Advanced provides enhanced DDoS detection, DRT access, and cost protection."
---

## Phase 1 â€” Triage

MUST:
- Check Shield subscription status: `aws shield describe-subscription`
- Get Shield policy details: `aws fms get-policy --policy-id <policy-id>`
- Check compliance status: `aws fms list-compliance-status --policy-id <policy-id>`
- List protected resources: `aws shield list-protections`

SHOULD:
- Verify Shield Advanced is enabled in the FMS admin account
- Check if member accounts have Shield Advanced enabled
- Verify resource types in policy scope are Shield-eligible
- Check for Shield Advanced subscription in member accounts

MAY:
- Review Shield Advanced attack history: `aws shield list-attacks --start-time <start> --end-time <end>`
- Check DRT (DDoS Response Team) access configuration

## Phase 2 â€” Remediate

MUST:
- Ensure Shield Advanced subscription is active
- Verify policy scope includes correct resource types
- Enable Shield Advanced on non-compliant resources

SHOULD:
- Configure health checks for protected resources
- Set up proactive engagement with DRT
- Enable Shield Advanced cost protection

MAY:
- Create CloudWatch dashboards for DDoS metrics
- Set up SNS notifications for Shield events

## Common Issues

- symptoms: "Shield Advanced not enabled on resources"
  diagnosis: "Shield subscription not active or resource type not in policy scope."
  resolution: "Activate Shield Advanced subscription. Verify resource types in policy scope."

- symptoms: "Shield policy shows non-compliant accounts"
  diagnosis: "Member accounts may not have Shield Advanced enabled."
  resolution: "FMS auto-remediation should enable Shield. Check for permission issues in member accounts."

## Output Format

```yaml
root_cause: "shield_policy_issue â€” <specific_cause>"
evidence:
  - type: subscription_status
    content: "<Shield Advanced subscription details>"
  - type: compliance_status
    content: "<policy compliance across accounts>"
severity: HIGH
mitigation:
  immediate: "Fix Shield Advanced subscription or policy scope"
  long_term: "Automate Shield Advanced deployment and monitoring"
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

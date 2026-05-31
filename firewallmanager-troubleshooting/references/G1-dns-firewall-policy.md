---
title: "G1 â€” DNS Firewall Policy Issues"
description: "Diagnose FMS DNS Firewall policy problems"
status: active
severity: HIGH
triggers:
  - "DNS Firewall policy error"
  - "DNS filtering not working"
  - "DNS Firewall not applied"
  - "domain blocking failed"
owner: devops-agent
objective: "Identify and resolve FMS DNS Firewall policy issues"
context: "FMS DNS Firewall policies distribute Route 53 Resolver DNS Firewall rule groups to member account VPCs. DNS Firewall filters DNS queries based on domain lists. Issues can arise from policy configuration, rule group priority conflicts, domain list errors, or VPC association failures."
---

## Phase 1 â€” Triage

MUST:
- Get DNS Firewall policy details: `aws fms get-policy --policy-id <policy-id>`
- Check compliance status: `aws fms list-compliance-status --policy-id <policy-id>`
- List DNS Firewall rule groups in member account: `aws route53resolver list-firewall-rule-groups`
- Check VPC associations: `aws route53resolver list-firewall-rule-group-associations`

SHOULD:
- Verify rule group rules: `aws route53resolver list-firewall-rules --firewall-rule-group-id <id>`
- Check domain list contents: `aws route53resolver get-firewall-domain-list --firewall-domain-list-id <id>`
- Verify rule group priority ordering
- Check for conflicts with existing DNS Firewall rule groups

MAY:
- Review Route 53 Resolver query logs for DNS filtering
- Check CloudWatch metrics for DNS Firewall

## Phase 2 â€” Remediate

MUST:
- Ensure DNS Firewall rule groups are properly configured
- Verify domain lists contain the correct domains
- Check rule group priority doesn't conflict with existing rules

SHOULD:
- Verify VPC associations are active
- Test DNS resolution for blocked domains
- Monitor DNS Firewall query logs after changes

MAY:
- Enable Route 53 Resolver query logging for troubleshooting
- Create CloudWatch dashboards for DNS Firewall metrics

## Common Issues

- symptoms: "DNS Firewall not blocking domains"
  diagnosis: "Rule group not associated with VPC or priority conflict."
  resolution: "Verify VPC association. Check rule group priority ordering."

- symptoms: "Legitimate domains being blocked"
  diagnosis: "Domain list too broad or rule action incorrect."
  resolution: "Review domain list entries. Adjust rules to use ALERT instead of BLOCK for testing."

## Output Format

```yaml
root_cause: "dns_firewall_policy â€” <specific_cause>"
evidence:
  - type: policy_config
    content: "<DNS Firewall policy configuration>"
  - type: rule_groups
    content: "<rule group and domain list details>"
severity: HIGH
mitigation:
  immediate: "Fix DNS Firewall policy configuration"
  long_term: "Implement DNS Firewall monitoring and domain list management"
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

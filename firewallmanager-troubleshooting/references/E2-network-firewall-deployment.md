---
title: "E2 â€” Network Firewall Deployment Issues"
description: "Diagnose Network Firewall endpoint deployment failures through FMS"
status: active
severity: HIGH
triggers:
  - "firewall endpoint not created"
  - "deployment failed"
  - "firewall not deployed"
  - "endpoint creation timeout"
owner: devops-agent
objective: "Identify and resolve Network Firewall deployment issues in FMS-managed VPCs"
context: "FMS deploys Network Firewall endpoints (ENIs) in member account VPCs. Deployment creates firewall subnets and endpoints. Issues can arise from subnet availability, IAM permissions, VPC limits, or deployment model configuration. Deployment takes several minutes per VPC."
---

## Phase 1 â€” Triage

MUST:
- Check firewall status in member account: `aws network-firewall describe-firewall --firewall-name <name>`
- Verify firewall endpoints: `aws network-firewall describe-firewall --firewall-name <name> --query 'FirewallStatus.SyncStates'`
- Check VPC subnet availability: `aws ec2 describe-subnets --filters Name=vpc-id,Values=<vpc-id>`
- Verify FMS compliance detail: `aws fms get-compliance-detail --policy-id <policy-id> --member-account <account-id>`

SHOULD:
- Check for VPC endpoint limits
- Verify FMS service-linked role permissions in member account
- Check CloudTrail for deployment events
- Verify the deployment model (distributed creates endpoints in each VPC)

MAY:
- Check Network Firewall service quotas
- Review VPC route tables for firewall integration

## Phase 2 â€” Remediate

MUST:
- Ensure VPC has available subnets for firewall endpoints
- Verify FMS SLR has permissions to create Network Firewall resources
- Wait for deployment to complete (several minutes per VPC)

SHOULD:
- Configure route tables to direct traffic through firewall endpoints
- Verify firewall endpoint health after deployment
- Monitor firewall metrics for traffic inspection

MAY:
- Implement centralized deployment model for hub-and-spoke architectures
- Set up automated route table management

## Common Issues

- symptoms: "Firewall endpoint stuck in CREATING state"
  diagnosis: "Subnet or ENI limits reached, or service issue."
  resolution: "Check VPC and subnet limits. Wait for deployment. Contact support if stuck."

- symptoms: "Traffic not being inspected after deployment"
  diagnosis: "Route tables not updated to direct traffic through firewall endpoints."
  resolution: "Update route tables in firewall and workload subnets to route through endpoints."

## Output Format

```yaml
root_cause: "network_firewall_deployment â€” <specific_cause>"
evidence:
  - type: firewall_status
    content: "<firewall and endpoint status>"
  - type: vpc_config
    content: "<VPC subnet and route table configuration>"
severity: HIGH
mitigation:
  immediate: "Fix deployment blockers and verify endpoint creation"
  long_term: "Automate deployment verification and route table management"
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

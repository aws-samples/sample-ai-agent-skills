---
title: "F1 — Hybrid Connectivity"
description: "Diagnose hybrid connectivity between on-premises mainframes and AWS M2"
status: active
severity: HIGH
triggers: ["cannot connect to mainframe", "VPN to mainframe failed", "Direct Connect issue", "data sync failed", "hybrid connectivity"]
owner: devops-agent
objective: "Resolve hybrid connectivity issues between on-premises mainframes and AWS M2 environments"
context: "During migration, hybrid connectivity is needed for data synchronization, parallel running, and gradual cutover. Connectivity uses VPN or Direct Connect. Data sync requires careful orchestration."
---

## Phase 1 — Triage

MUST:
- Check VPC connectivity: `aws ec2 describe-vpn-connections --query 'VpnConnections[*].{Id:VpnConnectionId,State:State,Type:Type}'`
- Verify M2 environment VPC configuration: `aws m2 get-environment --environment-id <env-id>`
- Check security group rules for on-premises connectivity: `aws ec2 describe-security-groups --group-ids <m2-sg-id> --query 'SecurityGroups[*].IpPermissions'`

SHOULD:
- Verify route table entries for on-premises CIDR ranges
- Check Direct Connect virtual interface status if using DX
- Test network connectivity from M2 environment to on-premises endpoints

MAY:
- Review VPC flow logs for blocked traffic
- Check Transit Gateway route tables if using TGW

## Phase 2 — Remediate

MUST:
- Ensure VPN or Direct Connect is active and routes are propagated
- Verify security groups allow traffic to/from on-premises mainframe IPs
- Confirm route tables include on-premises CIDR ranges

SHOULD:
- Test connectivity with specific ports used by mainframe protocols
- Verify DNS resolution for on-premises hostnames from M2 VPC

MAY:
- Implement redundant connectivity paths (VPN + Direct Connect)
- Configure network monitoring for hybrid link health

## Common Issues

- symptoms: "M2 application cannot connect to on-premises database during migration"
  diagnosis: "Security group or route table missing on-premises CIDR range."
  resolution: "Add on-premises CIDR to M2 environment security group inbound rules. Verify route table has route to on-premises via VPN/DX."

## Safety Ratings

```
safety_ratings:
  - "describe-vpn-connections: GREEN — read-only VPN status check"
  - "describe-security-groups: GREEN — read-only security group review"
  - "Security group modifications: YELLOW — changes network access controls"
  - "Route table modifications: RED — can affect all VPC routing"
```

## Escalation Conditions

- Complete loss of hybrid connectivity during active migration
- VPN or Direct Connect failures requiring infrastructure team involvement
- Firewall changes needed on the on-premises side
- Data synchronization failures due to connectivity issues

## Data Sensitivity

- VPN configuration: HIGH — reveals network security architecture
- On-premises network details: HIGH — reveals corporate network topology
- Security group rules: MEDIUM — reveals network access controls
- Route table entries: MEDIUM — reveals network routing architecture

## Prohibited Actions

- NEVER modify VPN or Direct Connect configuration without change management approval
- NEVER open overly broad security group rules for hybrid connectivity
- NEVER bypass corporate firewall rules for troubleshooting
- NEVER assume on-premises network changes can be made without coordination

## Phase 3 — Rollback

1. Revert security group changes to previous rules
2. Restore route table entries if modified
3. If VPN configuration was changed, revert to previous settings
4. Verify hybrid connectivity is restored after rollback
5. Document the connectivity issue and resolution

## Output Format

```yaml
root_cause: "hybrid_connectivity — <specific_cause>"
evidence:
  - type: connectivity_details
    content: "<VPN/DX status, security groups, route tables>"
severity: HIGH
mitigation:
  immediate: "Restore hybrid connectivity"
  long_term: "Implement redundant connectivity and monitoring"
```

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

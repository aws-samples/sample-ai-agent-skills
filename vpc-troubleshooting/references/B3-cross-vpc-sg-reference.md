---
title: "B3 — Cross-VPC Security Group References"
description: "Diagnose issues with security group rules referencing SGs in peered VPCs"
status: active
severity: MEDIUM
triggers:
  - "SG reference not working across peering"
  - "Cross-VPC SG rule not allowing traffic"
owner: devops-agent
objective: "Fix cross-VPC security group reference issues"
context: "SG-to-SG references work across VPC peering connections within the same region. They do NOT work across regions or through Transit Gateway. The referenced SG must be in a peered VPC with an active peering connection."
---

## Phase 1 — Triage

MUST:
- Verify the VPC peering connection is active: `aws ec2 describe-vpc-peering-connections`
- Check if the SG rule references a SG in the peered VPC (format: sg-xxx/vpc-xxx)
- Verify peering is in the same region (cross-region peering doesn't support SG references)
- Confirm the referenced SG still exists

SHOULD:
- Check if traffic is going through TGW instead of peering (SG references don't work via TGW)
- Verify DNS resolution across the peering connection
- Check route tables for peering routes

MAY:
- Test with CIDR-based rules to confirm SG reference is the issue
- Check if the peering connection was recently re-created (invalidates old SG references)

## Common Issues

- symptoms: "SG reference worked with peering, broke after migrating to TGW"
  diagnosis: "SG-to-SG references only work with VPC peering, not Transit Gateway."
  resolution: "Replace SG references with CIDR-based rules when using TGW."

- symptoms: "Cross-region peering, SG reference not working"
  diagnosis: "SG references are not supported across cross-region peering."
  resolution: "Use CIDR-based rules instead of SG references for cross-region."

## Safety Ratings

```
safety_ratings:
  - "describe-vpc-peering-connections: GREEN — read-only peering status check"
  - "describe-security-groups: GREEN — read-only SG rule inspection"
  - "describe-route-tables: GREEN — read-only route inspection"
  - "authorize-security-group-ingress (replace SG ref with CIDR): YELLOW — adds SG rule, recoverable by revoking"
  - "revoke-security-group-ingress (remove SG ref rule): YELLOW — removes SG rule, recoverable by re-adding"
```

## Escalation Conditions

- "Fix requires modifying production security group rules"
- "Fix requires migrating from SG references to CIDR-based rules across multiple SGs"
- "Fix involves Transit Gateway migration from VPC peering"
- "Multiple services rely on cross-VPC SG references"

## Data Sensitivity

- HIGH: security group rules (expose network architecture and cross-VPC access patterns)
- HIGH: VPC peering configurations (expose inter-VPC connectivity)
- MEDIUM: subnet CIDR ranges and VPC CIDR blocks

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER remove SG reference rules before adding equivalent CIDR-based rules"

## Phase 3 — Rollback

- If a CIDR-based rule was added to replace an SG reference: revoke it with `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol <proto> --port <port> --cidr <cidr>`
- If an SG reference rule was removed: re-add it with `aws ec2 authorize-security-group-ingress --group-id <sg-id> --protocol <proto> --port <port> --source-group <peer-sg-id>`
- Document all cross-VPC SG reference rules before making changes

## Output Format

```yaml
root_cause: "cross_vpc_sg_reference — <detail>"
evidence:
  - type: security_group
    content: "<SG rule and peering/TGW configuration>"
severity: MEDIUM
mitigation:
  immediate: "Replace SG reference with CIDR-based rule"
  long_term: "Use CIDR-based rules for TGW and cross-region connectivity"
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
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"

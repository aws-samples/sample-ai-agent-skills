---
title: "I3 — Secondary CIDR Issues"
description: "Diagnose issues with VPC secondary CIDR blocks"
status: active
severity: MEDIUM
triggers:
  - "Secondary CIDR routing"
  - "New CIDR not reachable"
  - "CIDR association failed"
owner: devops-agent
objective: "Fix secondary CIDR configuration and routing"
context: "VPCs support up to 5 IPv4 CIDRs (can request increase). Secondary CIDRs can be from RFC 1918 or 100.64.0.0/10 (CG-NAT range). Routes for secondary CIDRs are automatically added to the local route. Peering/TGW routes must be updated manually."
---

## Phase 1 — Triage

MUST:
- Check VPC CIDRs: `aws ec2 describe-vpcs --vpc-ids <vpc-id>` → CidrBlockAssociationSet
- Verify secondary CIDR state is 'associated'
- Check route tables include the secondary CIDR in local route
- Verify peering/TGW routes include the secondary CIDR

SHOULD:
- Check if the secondary CIDR overlaps with peered VPC CIDRs
- Verify subnets created from secondary CIDR have correct route table associations
- Check security groups and NACLs include the secondary CIDR range

MAY:
- Verify on-premises routing includes the secondary CIDR (VPN/DX)
- Check if the secondary CIDR is from the 100.64.0.0/10 range (restrictions apply)

## Common Issues

- symptoms: "Resources in secondary CIDR can't reach peered VPC"
  diagnosis: "Peered VPC route table only has route for primary CIDR, not secondary."
  resolution: "Add route for secondary CIDR → peering connection in the peer VPC."

- symptoms: "Secondary CIDR association failed"
  diagnosis: "CIDR overlaps with existing VPC CIDR, peering CIDR, or is invalid."
  resolution: "Choose a non-overlapping CIDR block."

## Safety Ratings

```
safety_ratings:
  - "describe-vpcs (CidrBlockAssociationSet): GREEN — read-only VPC CIDR inspection"
  - "describe-route-tables: GREEN — read-only route table inspection"
  - "describe-vpc-peering-connections: GREEN — read-only peering check"
  - "associate-vpc-cidr-block (add secondary CIDR): YELLOW — adds CIDR to VPC, recoverable by disassociating"
  - "create-subnet (create subnet from secondary CIDR): YELLOW — creates subnet, recoverable by deleting"
  - "create-route (add route for secondary CIDR): YELLOW — adds route, recoverable by deleting"
  - "disassociate-vpc-cidr-block: RED — removes CIDR, breaks resources in that range"
```

## Escalation Conditions

- "Fix requires adding secondary CIDR to production VPC"
- "Fix requires updating peering/TGW/VPN routes to include secondary CIDR"
- "Fix requires coordination with on-premises network team for VPN/DX route updates"
- "Secondary CIDR from 100.64.0.0/10 range — restrictions apply"

## Data Sensitivity

- HIGH: VPC CIDR blocks (expose network architecture and IP address plan)
- HIGH: route table entries (expose cross-VPC and hybrid routing)
- MEDIUM: peering configurations, TGW attachments, VPN route advertisements

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER disassociate a CIDR block that has active subnets or resources"

## Phase 3 — Rollback

- If a secondary CIDR was added: disassociate it with `aws ec2 disassociate-vpc-cidr-block --association-id <assoc-id>` (must delete all subnets in that CIDR first)
- If subnets were created from secondary CIDR: delete them with `aws ec2 delete-subnet --subnet-id <subnet-id>` (must be empty)
- If routes were added for secondary CIDR: delete them with `aws ec2 delete-route --route-table-id <rtb-id> --destination-cidr-block <secondary-cidr>`
- Document VPC CIDRs, peering routes, TGW routes, and VPN advertisements before changes

## Output Format

```yaml
root_cause: "secondary_cidr — <detail>"
evidence:
  - type: vpc
    content: "<VPC CIDRs and routing configuration>"
severity: MEDIUM
mitigation:
  immediate: "Update routes and security rules to include secondary CIDR"
  long_term: "Document all CIDRs, update peering/TGW/VPN routes when adding CIDRs"
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

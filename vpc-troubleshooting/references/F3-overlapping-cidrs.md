---
title: "F3 — Overlapping CIDRs"
description: "Diagnose connectivity issues caused by overlapping CIDR blocks across VPCs"
status: active
severity: HIGH
triggers:
  - "Routing conflict"
  - "Wrong destination for cross-VPC traffic"
  - "Peering rejected due to CIDR overlap"
owner: devops-agent
objective: "Identify CIDR overlaps and implement workarounds"
context: "VPC peering does NOT allow overlapping CIDRs. TGW can connect VPCs with overlapping CIDRs but routing becomes ambiguous. Overlapping CIDRs are a common issue in enterprise environments with legacy IP planning."
---

## Phase 1 — Triage

MUST:
- List CIDRs for all involved VPCs: `aws ec2 describe-vpcs --vpc-ids <ids>` → CidrBlockAssociationSet
- Check for overlapping ranges between VPCs that need to communicate
- Identify if peering was rejected due to overlap
- Check if secondary CIDRs were added that create overlap

SHOULD:
- Map out the full IP address plan across all connected VPCs
- Check if PrivateLink could be used instead of peering/TGW for specific services
- Verify if the overlap is partial or complete

MAY:
- Evaluate re-IPing one of the VPCs (add non-overlapping secondary CIDR)
- Consider AWS PrivateLink as an alternative to direct routing

## Common Issues

- symptoms: "VPC peering creation fails with CIDR overlap error"
  diagnosis: "Both VPCs use the same or overlapping CIDR (e.g., both use 10.0.0.0/16)."
  resolution: "Add a non-overlapping secondary CIDR to one VPC and use that for peering routes."

- symptoms: "TGW routes to two VPCs with same CIDR, traffic goes to wrong VPC"
  diagnosis: "TGW can't distinguish between two VPCs with the same CIDR."
  resolution: "Use PrivateLink for service-to-service communication, or re-IP one VPC."

## Safety Ratings

```
safety_ratings:
  - "describe-vpcs: GREEN — read-only VPC CIDR inspection"
  - "describe-vpc-peering-connections: GREEN — read-only peering status check"
  - "describe-transit-gateway-attachments: GREEN — read-only TGW attachment listing"
  - "associate-vpc-cidr-block (add secondary CIDR): YELLOW — adds CIDR to VPC, recoverable by disassociating"
  - "create-vpc-endpoint (add PrivateLink): YELLOW — creates endpoint, recoverable by deleting"
  - "disassociate-vpc-cidr-block: RED — removes secondary CIDR, can break resources in that range"
```

## Escalation Conditions

- "Fix requires adding secondary CIDR to production VPC"
- "Fix requires re-IPing resources — significant migration effort"
- "Multiple VPCs affected by overlapping CIDRs"
- "Fix involves PrivateLink setup — requires service provider coordination"

## Data Sensitivity

- HIGH: VPC CIDR ranges (expose network architecture and IP address plan)
- HIGH: VPC peering configurations (expose inter-VPC connectivity)
- MEDIUM: TGW attachment configurations, PrivateLink endpoints

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER remove a CIDR block that has active subnets or resources"

## Phase 3 — Rollback

- If a secondary CIDR was added: disassociate it with `aws ec2 disassociate-vpc-cidr-block --association-id <assoc-id>` (must delete subnets in that CIDR first)
- If a PrivateLink endpoint was created: delete it with `aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <vpce-id>`
- If routes were added for the new CIDR: delete them with `aws ec2 delete-route --route-table-id <rtb-id> --destination-cidr-block <cidr>`
- Document all VPC CIDRs, peering connections, and TGW attachments before changes

## Output Format

```yaml
root_cause: "overlapping_cidrs — <VPC CIDRs>"
evidence:
  - type: vpc
    content: "<VPC IDs and their overlapping CIDRs>"
severity: HIGH
mitigation:
  immediate: "Use PrivateLink or add non-overlapping secondary CIDRs"
  long_term: "Implement centralized IP address management (IPAM)"
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

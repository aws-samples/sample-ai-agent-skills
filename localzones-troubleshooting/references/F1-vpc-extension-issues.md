---
title: "F1 — VPC Extension Issues"
description: "Diagnose VPC extension and multi-zone architecture issues"
status: active
severity: HIGH
triggers: ["VPC extension error", "cannot extend VPC", "Local Zone VPC", "multi-zone networking"]
owner: devops-agent
objective: "Resolve VPC extension issues when adding Local Zone subnets"
context: "Local Zones are accessed by extending existing VPCs with subnets in the Local Zone. You cannot create a VPC in a Local Zone. VPC features like flow logs, DNS, and DHCP options apply to Local Zone subnets."
---

## Phase 1 — Triage

MUST:
- Check VPC configuration: `aws ec2 describe-vpcs --vpc-ids <vpc-id> --query 'Vpcs[0].{CidrBlock:CidrBlock,State:State,DhcpOptionsId:DhcpOptionsId}'`
- List subnets across all zones: `aws ec2 describe-subnets --filters "Name=vpc-id,Values=<vpc-id>" --query 'Subnets[*].{SubnetId:SubnetId,CidrBlock:CidrBlock,AZ:AvailabilityZone,State:State}'`
- Check VPC CIDR associations: `aws ec2 describe-vpcs --vpc-ids <vpc-id> --query 'Vpcs[0].CidrBlockAssociationSet'`

SHOULD:
- Verify available CIDR space for new Local Zone subnets
- Check VPC DNS settings: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`
- Review DHCP options set for DNS configuration

MAY:
- Check VPC peering connections that may need route updates
- Review Transit Gateway attachments for Local Zone routing
- Check VPC endpoint configurations

## Phase 2 — Remediate

MUST:
- Ensure VPC has sufficient CIDR space for Local Zone subnets
- Add secondary CIDR if needed: `aws ec2 associate-vpc-cidr-block --vpc-id <vpc-id> --cidr-block <cidr>`
- Create subnet in Local Zone with available CIDR range

SHOULD:
- Update route tables for new Local Zone subnets
- Configure VPC flow logs to include Local Zone traffic
- Update VPC peering routes if applicable

MAY:
- Configure Transit Gateway for Local Zone connectivity
- Set up VPC endpoints for AWS service access from Local Zone

## Safety Ratings

```
safety_ratings:
  - "describe-vpcs: GREEN — read-only VPC details"
  - "describe-subnets: GREEN — read-only subnet enumeration"
  - "associate-vpc-cidr-block: YELLOW — expands VPC address space"
  - "create-subnet: YELLOW — creates new network resource"
  - "disassociate-vpc-cidr-block: RED — removes CIDR, may break subnets"
```

## Escalation Conditions

- VPC CIDR exhausted and cannot add secondary CIDR (limit of 5 CIDRs per VPC)
- VPC peering or Transit Gateway changes require cross-account coordination
- VPC extension conflicts with existing network architecture
- DHCP options changes affect all subnets in the VPC

## Data Sensitivity

- VPC CIDR configuration: MEDIUM — reveals network address space
- Subnet layout: MEDIUM — reveals network segmentation strategy
- VPC peering details: MEDIUM — reveals cross-VPC connectivity
- Transit Gateway configuration: MEDIUM — reveals network topology

## Prohibited Actions

- NEVER disassociate a VPC CIDR block that has active subnets
- NEVER modify DHCP options without understanding impact on all VPC resources
- NEVER create overlapping CIDRs between VPC and peered VPCs
- NEVER extend VPC to Local Zone without network architecture review

## Phase 3 — Rollback

1. If subnet was created with wrong CIDR, delete it (if no resources): `aws ec2 delete-subnet --subnet-id <id>`
2. If secondary CIDR was added incorrectly, remove subnets first then disassociate: `aws ec2 disassociate-vpc-cidr-block --association-id <id>`
3. If route table was modified, restore previous routes
4. Verify all VPC connectivity is intact after rollback

## Common Issues

- symptoms: "Cannot create subnet — VPC CIDR space exhausted"
  diagnosis: "All VPC CIDR ranges are allocated to existing subnets."
  resolution: "Add secondary CIDR to VPC. Plan CIDR allocation to reserve space for Local Zone subnets."

- symptoms: "Local Zone subnet cannot reach VPC peered resources"
  diagnosis: "VPC peering route table does not include Local Zone subnet CIDR."
  resolution: "Update peering route tables to include Local Zone subnet CIDR ranges."

## Output Format

```yaml
root_cause: "vpc_extension — <specific_cause>"
evidence:
  - type: vpc_config
    content: "<VPC CIDR and subnet configuration>"
severity: HIGH
mitigation:
  immediate: "Fix CIDR allocation or routing"
  long_term: "Implement CIDR planning for multi-zone VPC architecture"
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

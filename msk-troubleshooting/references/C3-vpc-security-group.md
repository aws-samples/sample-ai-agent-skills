---
title: "C3 — MSK VPC and Security Group Issues"
description: "Diagnose and resolve VPC networking and security group configuration issues"
status: active
severity: HIGH
triggers:
  - "security group"
  - "VPC"
  - "subnet"
  - "network"
  - "ENI"
  - "cross-VPC"
  - "peering"
owner: devops-agent
objective: "Resolve VPC networking issues preventing client-to-broker communication"
context: "MSK brokers are deployed in private subnets within a VPC. Security groups control inbound/outbound traffic. Clients must have network path to brokers on all required ports (9092-9098). Cross-VPC access requires peering, Transit Gateway, or PrivateLink. MSK creates ENIs in the specified subnets. Subnet sizing, route tables, and NACLs can all affect connectivity."
---

## Phase 1 — Triage

MUST:
- Check broker security group: `aws kafka describe-cluster-v2 --cluster-arn <cluster-arn> --query 'ClusterInfo.Provisioned.BrokerNodeGroupInfo.SecurityGroups'`
- Check security group inbound rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[*].IpPermissions'`
- Check broker subnets: `aws kafka describe-cluster-v2 --cluster-arn <cluster-arn> --query 'ClusterInfo.Provisioned.BrokerNodeGroupInfo.ClientSubnets'`
- Check subnet details: `aws ec2 describe-subnets --subnet-ids <subnet-ids> --query 'Subnets[*].{SubnetId:SubnetId,VpcId:VpcId,AZ:AvailabilityZone,CIDR:CidrBlock,AvailableIPs:AvailableIpAddressCount}'`

SHOULD:
- Check client security group outbound rules
- Check subnet route tables: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Check NACLs on broker subnets: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`
- Check VPC peering connections (if cross-VPC): `aws ec2 describe-vpc-peering-connections --filters Name=status-code,Values=active`

MAY:
- Check VPC flow logs for rejected traffic
- Check ENIs created by MSK: `aws ec2 describe-network-interfaces --filters Name=description,Values="*MSK*" Name=vpc-id,Values=<vpc-id>`
- Check Transit Gateway routes (if using TGW)

## Phase 2 — Remediate

MUST:
- If security group blocks: add inbound rules for TCP ports 9092-9098 from client CIDR or security group
- If cross-VPC without connectivity: set up VPC peering, Transit Gateway, or PrivateLink
- If route tables missing routes: add routes for client CIDR via peering connection or TGW
- If NACLs blocking: add allow rules for broker ports (NACLs are stateless — need both inbound and outbound)

SHOULD:
- Allow all broker ports (9092, 9093, 9094, 9096, 9098) in security groups
- Ensure DNS resolution works across VPCs (enable DNS resolution on peering connection)
- Use security group references instead of CIDR blocks when possible

MAY:
- Enable multi-VPC connectivity (MSK managed feature)
- Set up VPC endpoints for MSK API calls
- Consider AWS PrivateLink for cross-account access

## Common Issues

- symptoms: "Client connects to bootstrap but fails on metadata fetch"
  diagnosis: "Security group only allows bootstrap port but not individual broker ports."
  resolution: "Open all broker ports (9092-9098) in the security group, not just the bootstrap port."

- symptoms: "Cross-VPC client cannot resolve broker DNS names"
  diagnosis: "VPC peering DNS resolution not enabled."
  resolution: "Enable DNS resolution on the VPC peering connection. Use Route 53 Resolver if needed."

- symptoms: "Intermittent connectivity from client in different AZ"
  diagnosis: "Route table for client subnet missing route to broker subnet CIDR."
  resolution: "Add route to broker VPC CIDR via peering connection or Transit Gateway in client route table."

## Output Format

```yaml
root_cause: "vpc_security_group — <specific_cause>"
evidence:
  - type: security_groups
    content: "<inbound/outbound rules>"
  - type: network_config
    content: "<subnets, route tables, NACLs>"
  - type: connectivity_test
    content: "<network test results>"
severity: HIGH
mitigation:
  immediate: "Fix security group rules or network routing"
  long_term: "Document network architecture, automate security group management"
```


## Safety Ratings
```
safety_ratings:
  - "Check broker security group: GREEN — read-only API call"
  - "Check security group inbound rules: GREEN — read-only network inspection"
  - "Check subnet details and route tables: GREEN — read-only network inspection"
  - "Add security group inbound rules: YELLOW — changes network access controls"
  - "Set up VPC peering: YELLOW — creates cross-VPC network path"
  - "Modify NACLs: YELLOW — stateless rules, easy to misconfigure and block traffic"
  - "Enable multi-VPC connectivity: YELLOW — expands cluster network access surface"
```

## Escalation Conditions
- Cluster serves production event streaming
- VPC networking changes affect multiple services beyond MSK
- Cross-VPC connectivity requires coordination with networking team
- Security group changes may impact compliance requirements
- NACL modifications risk blocking traffic for other services in the subnet

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "VPC IDs and CIDR ranges: network topology"
    - "Security group rules: access control configuration"
    - "Subnet IDs and route tables: infrastructure layout"
    - "VPC peering connection details: cross-network relationships"
  handling: "Do not expose VPC topology or security group configurations externally. Mask CIDR ranges in shared reports."
```

## Prohibited Actions
- NEVER suggest deleting topics with active consumers
- NEVER suggest reducing broker count
- NEVER open security groups to 0.0.0.0/0 for broker ports
- NEVER remove security group rules without verifying impact on all services using that group
- NEVER modify NACLs without adding both inbound and outbound rules (NACLs are stateless)
- NEVER delete VPC peering connections without confirming no other services depend on them

## Phase 3 — Rollback
- If security group rules were added: remove the specific rules that were added, preserving existing rules
- If VPC peering was created: delete peering connection and remove route table entries in both VPCs
- If route table entries were modified: revert to previous routing configuration
- If NACLs were modified: restore previous NACL rules (both inbound and outbound)
- If multi-VPC connectivity was enabled: disable the feature if not needed (may require AWS Support)
- If Transit Gateway routes were added: remove the specific routes from TGW route tables

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest removing authentication"
  - "NEVER suggest public access to brokers"

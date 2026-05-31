---
title: "D1 — VPC Configuration Issues"
description: "Diagnose VPC and networking problems for ParallelCluster"
status: active
severity: HIGH
triggers:
  - "VPC parallelcluster"
  - "subnet configuration"
  - "network connectivity"
  - "head node internet"
owner: devops-agent
objective: "Resolve VPC, subnet, and network connectivity issues for ParallelCluster"
context: "ParallelCluster requires specific VPC configuration: the head node needs internet access or VPC endpoints, compute nodes need access to the head node and shared storage. Public and private subnet configurations are supported."
---

## Phase 1 — Triage

MUST:
- Check subnet configuration: `aws ec2 describe-subnets --subnet-ids <head-subnet> <compute-subnet>`
- Verify route tables: `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<subnet-id>"`
- Check NAT gateway (for private subnets): `aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=<vpc-id>"`
- Verify security groups: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check VPC endpoints if no internet access: `aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=<vpc-id>"`

SHOULD:
- Verify head node can reach AWS API endpoints
- Check DNS resolution is enabled on the VPC
- Verify CIDR ranges don't conflict

MAY:
- Check VPC flow logs for dropped traffic
- Review NACLs for restrictive rules

## Phase 2 — Remediate

MUST:
- If no internet: add NAT gateway route or create VPC endpoints for required services
- If security groups blocking: allow inbound SSH (22), Slurm (6817-6819), NFS (2049), Lustre (988)
- If DNS failing: enable DNS support and DNS hostnames on VPC

SHOULD:
- Use separate subnets for head node (public) and compute (private)
- Create VPC endpoints for S3, CloudWatch, CloudFormation, EC2, and logs
- Document the network architecture

MAY:
- Implement VPC flow logs for troubleshooting
- Use Transit Gateway for multi-VPC architectures

## Common Issues

- symptoms: "Cluster creation fails — head node cannot reach AWS APIs"
  diagnosis: "Head node subnet has no internet access and no VPC endpoints."
  resolution: "Add NAT gateway route or create VPC endpoints for required AWS services."

- symptoms: "Compute nodes cannot reach head node"
  diagnosis: "Security groups not allowing Slurm ports between head and compute."
  resolution: "Allow ports 6817-6819 (Slurm) between head and compute security groups."

## Output Format

```yaml
root_cause: "vpc_config — <specific_cause>"
evidence:
  - type: subnet_config
    content: "<subnet and route table details>"
  - type: security_groups
    content: "<security group rules>"
severity: HIGH
mitigation:
  immediate: "Fix routing, security groups, or VPC endpoints"
  long_term: "Implement proper VPC architecture for ParallelCluster"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — VPC misconfigurations can expose cluster resources
- Service Disruption Risk: HIGH — network issues can make clusters inaccessible
- Compliance Impact: HIGH — VPC configuration affects network security posture

## Escalation Conditions
- Cluster loses network connectivity
- Security group changes block inter-node communication
- NAT gateway failure prevents compute node provisioning
- DNS resolution failures affect cluster operations
- VPC peering changes break cluster integrations

## Data Sensitivity
- VPC CIDR ranges — INTERNAL
- Security group rules — INTERNAL
- Route table configurations — INTERNAL
- NAT gateway details — INTERNAL
- Never expose internal cluster network topology

## Prohibited Actions
- Do NOT open cluster security groups to 0.0.0.0/0
- Do NOT modify VPC route tables without understanding cluster traffic
- Do NOT delete NAT gateways required by compute nodes
- Do NOT place head nodes in public subnets without justification
- Do NOT disable VPC flow logs during investigations
- Do NOT modify inter-node security group rules during active jobs

## Phase 3 — Rollback
- Restore previous security group rules
- Revert route table changes
- Re-create NAT gateways if deleted
- Restore DNS configurations
- Verify cluster connectivity after network changes
- Document VPC changes and their impact on cluster operations

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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

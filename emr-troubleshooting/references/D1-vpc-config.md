---
title: "D1 — VPC Configuration Issues"
description: "Diagnose VPC and networking issues for EMR clusters"
status: active
severity: HIGH
triggers:
  - "VPC configuration error"
  - "subnet issue"
  - "no internet access"
  - "EMR networking"
owner: devops-agent
objective: "Identify and resolve VPC configuration issues for EMR"
context: "EMR clusters run in VPC subnets. They need internet access (NAT gateway or VPC endpoints) for service API calls. Private subnets require VPC endpoints for S3 and EMR. Subnet must have sufficient available IPs for all cluster nodes."
---

## Phase 1 — Triage

MUST:
- Check subnet configuration: `aws ec2 describe-subnets --subnet-ids <subnet-id> --query 'Subnets[*].{Id:SubnetId,AZ:AvailabilityZone,AvailableIPs:AvailableIpAddressCount,VpcId:VpcId}'`
- Verify route table has internet access: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Check for S3 VPC endpoint: `aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id> Name=service-name,Values=com.amazonaws.*.s3`
- Verify DNS resolution is enabled on VPC

SHOULD:
- Check for EMR VPC endpoint (for private subnets)
- Verify NAT gateway is functional (for private subnets)
- Check NACLs for blocking rules
- Verify sufficient available IPs in the subnet

MAY:
- Check for VPC peering or Transit Gateway connectivity
- Verify DNS resolution for external services

## Phase 2 — Remediate

MUST:
- Ensure subnet has internet access (NAT or VPC endpoints)
- Create S3 gateway endpoint for the VPC
- Ensure sufficient available IPs in the subnet

SHOULD:
- Create EMR interface endpoint for private subnets
- Configure DNS resolution on the VPC
- Use multiple subnets across AZs for availability

MAY:
- Implement VPC flow logs for network debugging
- Create standardized VPC templates for EMR

## Common Issues

- symptoms: "Cluster fails to provision in private subnet"
  diagnosis: "No internet access and missing VPC endpoints."
  resolution: "Add NAT gateway or create S3 and EMR VPC endpoints."

- symptoms: "Insufficient IP addresses"
  diagnosis: "Subnet CIDR too small for the cluster size."
  resolution: "Use a larger subnet or reduce cluster size."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure subnet has internet access (NAT or endpoints) | YELLOW | Network infrastructure change — verify routing |
| Create S3 gateway endpoint | GREEN | Adds network path — non-destructive |
| Create EMR interface endpoint | GREEN | Adds network path — non-destructive |
| Configure DNS resolution on VPC | YELLOW | VPC-level setting — affects all resources in VPC |
| Use multiple subnets across AZs | GREEN | Improves availability — non-destructive |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- VPC changes affect other services in the same VPC
- Network configuration is managed by a separate networking team

## Data Sensitivity

- **Classification: HIGH**
- Subnet and VPC configuration reveals network architecture
- Route tables expose connectivity to external networks and services
- VPC endpoint configuration reveals service access patterns
- DNS settings may reveal internal service discovery architecture

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest opening all ports in security groups or NACLs to resolve connectivity
- **NEVER** suggest placing EMR clusters in public subnets without security justification

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Created S3 VPC endpoint | Delete the VPC endpoint if not needed |
| Created EMR interface endpoint | Delete the interface endpoint if not needed |
| Modified route tables | Revert route table entries to previous configuration |
| Changed DNS resolution settings | Revert VPC DNS settings to previous state |

## Output Format

```yaml
root_cause: "vpc_config — <specific_cause>"
evidence:
  - type: subnet
    content: "<subnet configuration>"
  - type: routing
    content: "<route table and internet access>"
severity: HIGH
mitigation:
  immediate: "Fix VPC networking configuration"
  long_term: "Create standardized VPC templates for EMR deployments"
```

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
  - "NEVER suggest disabling encryption"
  - "NEVER suggest public subnet placement for clusters"
  - "NEVER suggest removing Kerberos authentication"

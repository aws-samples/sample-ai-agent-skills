---
title: "F1 — VPC Connectivity"
description: "Diagnose Managed Flink VPC connectivity issues"
status: active
severity: HIGH
triggers: ["VPC error", "cannot access Kinesis from VPC", "NAT Gateway", "VPC endpoint", "network timeout"]
owner: devops-agent
objective: "Resolve VPC connectivity issues including NAT Gateway, VPC endpoints, security groups, and subnet configuration"
context: "VPC-connected applications run in customer VPC subnets. They need NAT Gateway for internet and VPC endpoints for AWS services."
---

## Phase 1 — Triage

MUST:
- Check VPC configuration: `aws kinesisanalyticsv2 describe-application --application-name <app-name> --query 'ApplicationDetail.ApplicationConfigurationDescription.VpcConfigurationDescriptions'`
- Verify subnets have available IP addresses: `aws ec2 describe-subnets --subnet-ids <subnet-ids> --query 'Subnets[*].{Id:SubnetId,AvailableIPs:AvailableIpAddressCount,AZ:AvailabilityZone}'`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-ids>`

SHOULD:
- Verify NAT Gateway exists for internet access: `aws ec2 describe-nat-gateways --filter Name=vpc-id,Values=<vpc-id>`
- Check VPC endpoints for AWS services: `aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id>`

MAY:
- Check route tables for correct routing: `aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id>`
- Review VPC Flow Logs for denied traffic

## Phase 2 — Remediate

MUST:
- Create VPC endpoints for required AWS services (Kinesis, S3, CloudWatch)
- Configure NAT Gateway if internet access is needed
- Ensure security groups allow outbound traffic to required destinations

SHOULD:
- Use at least 2 subnets in different AZs for high availability
- Ensure subnets have sufficient IP addresses (Flink uses ENIs)

MAY:
- Configure VPC endpoint policies for least-privilege access
- Set up VPC Flow Logs for network troubleshooting

## Safety Ratings

```
safety_ratings:
  - "describe-application: GREEN — read-only application details"
  - "ec2 describe-subnets: GREEN — read-only subnet details"
  - "ec2 describe-security-groups: GREEN — read-only security group rules"
  - "ec2 create-vpc-endpoint: YELLOW — creates network infrastructure"
```

## Escalation Conditions

- VPC endpoint creation requires networking team approval
- Security group changes affect other services in the VPC
- Subnet IP address exhaustion requires CIDR expansion
- Cross-account VPC access is required for source/sink

## Data Sensitivity

- `aws ec2 describe-subnets`: MEDIUM — reveals VPC networking configuration
- `aws ec2 describe-security-groups`: MEDIUM — reveals network access rules
- VPC Flow Logs: HIGH — reveal network traffic patterns

## Prohibited Actions

- NEVER open security groups to 0.0.0.0/0 for troubleshooting
- NEVER delete VPC endpoints used by other services
- NEVER modify route tables without understanding impact on other workloads
- NEVER use public subnets for Managed Flink applications

## Phase 3 — Rollback

1. If security group rules were added, remove them
2. If VPC endpoints were created for testing, delete them if not needed
3. Restore original route table configuration if modified
4. Verify other services in the VPC are not affected
5. Document the correct VPC configuration for the application

## Common Issues

- symptoms: "Application cannot connect to Kinesis stream, timeout errors in logs"
  diagnosis: "VPC-connected application has no VPC endpoint or NAT Gateway for Kinesis access."
  resolution: "Create a VPC endpoint for com.amazonaws.<region>.kinesis or configure NAT Gateway for internet access."

## Output Format

```yaml
root_cause: "vpc_connectivity — <specific_cause>"
evidence:
  - type: vpc_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix VPC connectivity"
  long_term: "Document VPC requirements"
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

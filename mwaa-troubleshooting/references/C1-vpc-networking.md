---
title: "C1 — MWAA VPC Networking Issues"
description: "Diagnose VPC networking problems affecting MWAA environments"
status: active
severity: HIGH
triggers:
  - "network error"
  - "connectivity issue"
  - "NAT gateway"
  - "subnet"
  - "security group"
  - "cannot reach"
owner: devops-agent
objective: "Identify and resolve VPC networking issues preventing MWAA from functioning correctly"
context: "MWAA requires private subnets with NAT gateway for outbound internet access. Security groups must allow self-referencing inbound traffic. Network issues affect package installation, DAG sync, AWS API calls, and external service connectivity."
---

## Phase 1 — Triage

MUST:
- Check environment VPC config: `aws mwaa get-environment --name <env-name> --query 'Environment.NetworkConfiguration'`
- Verify subnets are private: `aws ec2 describe-subnets --subnet-ids <subnet-ids> --query 'Subnets[*].{SubnetId:SubnetId,AZ:AvailabilityZone,MapPublicIp:MapPublicIpOnLaunch}'`
- Check NAT gateway status: `aws ec2 describe-nat-gateways --filter Name=vpc-id,Values=<vpc-id> --query 'NatGateways[*].{Id:NatGatewayId,State:State}'`
- Check route tables for NAT route: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id> --query 'RouteTables[*].Routes'`
- Check security groups: `aws ec2 describe-security-groups --group-ids <sg-ids>`

SHOULD:
- Verify security group has self-referencing inbound rule (all traffic from same SG)
- Check outbound rules allow HTTPS (443) to 0.0.0.0/0
- Verify VPC DNS settings: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`
- Check VPC endpoints if using them instead of NAT: `aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id>`

MAY:
- Check VPC flow logs for rejected traffic: `aws logs filter-log-events --log-group-name <flow-log-group> --filter-pattern "REJECT"`
- Verify NACLs are not blocking traffic: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`

## Phase 2 — Remediate

MUST:
- Ensure subnets are private (MapPublicIpOnLaunch=false)
- Create or fix NAT gateway in a public subnet with route from private subnets
- Add self-referencing inbound rule to security group

SHOULD:
- Enable DNS support and DNS hostnames on the VPC
- Allow outbound HTTPS (443) in security groups
- Use at least two subnets in different AZs

MAY:
- Add VPC endpoints for S3, CloudWatch Logs, SQS, and KMS to reduce NAT costs
- Enable VPC flow logs for ongoing network troubleshooting

## Common Issues

- symptoms: "Environment creation fails with network error"
  diagnosis: "Subnets are public or NAT gateway is missing."
  resolution: "Use private subnets with NAT gateway. Verify route tables."

- symptoms: "Tasks fail with connection timeout to AWS services"
  diagnosis: "NAT gateway is down or route table missing NAT route."
  resolution: "Check NAT gateway state and route table entries."

- symptoms: "requirements.txt packages fail to install"
  diagnosis: "No outbound internet access for PyPI downloads."
  resolution: "Verify NAT gateway is active and route tables are correct."

## Output Format

```yaml
root_cause: "vpc_networking_issue — <specific_cause>"
evidence:
  - type: vpc_config
    content: "<subnets, NAT gateway, route tables>"
  - type: security_groups
    content: "<inbound and outbound rules>"
  - type: connectivity_test
    content: "<flow logs or error messages>"
severity: HIGH
mitigation:
  immediate: "Fix the identified networking issue (NAT, routes, security groups)"
  long_term: "Use IaC to manage VPC configuration with validated MWAA networking"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — VPC misconfigurations can expose internal resources
- Service Disruption Risk: HIGH — networking issues can make environments completely inaccessible
- Compliance Impact: HIGH — VPC configuration directly affects network security posture

## Escalation Conditions
- Environment loses all network connectivity
- NAT gateway failure affects multiple MWAA environments
- VPC peering or Transit Gateway changes break MWAA connectivity
- Security group changes lock out all access to the web server
- DNS resolution failures prevent MWAA from reaching AWS services

## Data Sensitivity
- VPC CIDR ranges and subnet configurations — INTERNAL
- Security group rules — INTERNAL
- NAT gateway and route table configurations — INTERNAL
- VPC endpoint configurations — INTERNAL
- Network flow logs — CONFIDENTIAL
- Never expose internal IP addresses externally

## Prohibited Actions
- Do NOT open MWAA security groups to 0.0.0.0/0
- Do NOT remove NAT gateway without providing alternative internet access
- Do NOT modify VPC route tables without understanding MWAA traffic patterns
- Do NOT delete VPC endpoints required by MWAA services
- Do NOT place MWAA in public subnets
- Do NOT disable VPC flow logs on MWAA subnets during active investigations

## Phase 3 — Rollback
- Restore previous security group rules from documented baseline
- Re-create NAT gateway if accidentally deleted
- Revert route table changes to restore MWAA connectivity
- Restore VPC endpoint configurations if removed
- Verify MWAA environment health after network changes
- Document all VPC changes and their impact on MWAA connectivity

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
  - "NEVER suggest disabling webserver authentication"
  - "NEVER suggest public network access mode for production"
  - "NEVER suggest overly broad Airflow execution role"

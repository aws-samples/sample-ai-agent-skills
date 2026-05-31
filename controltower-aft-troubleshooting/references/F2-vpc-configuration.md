---
title: "F2 — VPC Configuration"
description: "Diagnose and resolve VPC configuration issues affecting AFT CodeBuild and Lambda"
status: active
severity: HIGH
triggers: ["VPC connectivity error", "CodeBuild timeout in VPC", "NAT gateway issue", "Lambda VPC error"]
owner: devops-agent
objective: "Ensure AFT components have proper network connectivity through VPC configuration"
context: "AFT CodeBuild projects and Lambda functions may run in a VPC for security. VPC configuration requires NAT gateway for internet access, proper subnet routing, and security group rules. VPC misconfiguration is a common cause of silent timeouts."
---

## Phase 1 — Triage

MUST:
- Check CodeBuild project VPC configuration: `aws codebuild batch-get-projects --names <project-name> --query 'projects[0].vpcConfig'`
- Verify NAT gateway exists and is active: `aws ec2 describe-nat-gateways --filter 'Name=vpc-id,Values=<vpc-id>' --query 'NatGateways[*].{Id:NatGatewayId,State:State,SubnetId:SubnetId}'`
- Check route tables for private subnets: `aws ec2 describe-route-tables --filters 'Name=association.subnet-id,Values=<subnet-id>'`

SHOULD:
- Verify security group outbound rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[0].IpPermissionsEgress'`
- Check subnet available IP addresses: `aws ec2 describe-subnets --subnet-ids <subnet-id> --query 'Subnets[0].AvailableIpAddressCount'`
- Verify VPC endpoints if configured for AWS services

MAY:
- Check VPC flow logs for dropped traffic
- Review Lambda function VPC configuration if applicable
- Test connectivity from an EC2 instance in the same subnet

## Phase 2 — Remediate

MUST:
- Ensure private subnets have a route to NAT gateway (0.0.0.0/0 → NAT gateway)
- Verify security groups allow outbound HTTPS (port 443) to 0.0.0.0/0
- Ensure subnets have sufficient available IP addresses

SHOULD:
- Configure VPC endpoints for S3, DynamoDB, and other AWS services used by AFT
- Verify NAT gateway is in a public subnet with an internet gateway route
- Check for NACL rules that may block traffic

MAY:
- Set up redundant NAT gateways across availability zones
- Implement VPC flow log monitoring for connectivity issues

## Safety Ratings

```
safety_ratings:
  - "ec2 describe-nat-gateways: GREEN — read-only NAT gateway check"
  - "ec2 describe-route-tables: GREEN — read-only routing check"
  - "ec2 describe-security-groups: GREEN — read-only security group check"
  - "Route table modification: YELLOW — changes network routing"
```

## Escalation Conditions

- VPC is managed by a network team and routing changes require approval
- NAT gateway is at capacity or experiencing throttling
- VPC endpoint service is not available in the region
- Subnet IP address exhaustion requires VPC CIDR expansion

## Data Sensitivity

- VPC configuration: MEDIUM — reveals network architecture
- Security group rules: MEDIUM — reveals network access patterns
- Route tables: MEDIUM — reveals network routing topology
- VPC flow logs: HIGH — contain source/destination IP addresses and traffic patterns

## Prohibited Actions

- NEVER open security groups to 0.0.0.0/0 for inbound traffic
- NEVER delete NAT gateways without understanding the impact on other services
- NEVER modify route tables without a rollback plan
- NEVER remove VPC endpoints that other services depend on

## Phase 3 — Rollback

1. If route table changes caused connectivity issues, revert to previous routes
2. If security group rules were modified, revert to previous rules
3. If NAT gateway was replaced, update route tables to point to the new NAT gateway
4. Document all VPC configuration changes
5. Verify all AFT components have connectivity after changes

## Common Issues

- symptoms: "CodeBuild times out during DOWNLOAD_SOURCE or INSTALL phase"
  diagnosis: "CodeBuild in VPC cannot reach the internet. Missing NAT gateway or incorrect routing."
  resolution: "Verify NAT gateway in public subnet. Check private subnet route table has 0.0.0.0/0 → NAT gateway."

- symptoms: "Lambda function times out when calling AWS APIs"
  diagnosis: "Lambda in VPC without NAT gateway or VPC endpoints for AWS services."
  resolution: "Add NAT gateway route or create VPC endpoints for required AWS services (STS, DynamoDB, S3)."

## Output Format

```yaml
root_cause: "vpc_configuration — <specific_cause>"
evidence:
  - type: vpc_error
    content: "<VPC, subnet, route table, and security group details>"
severity: HIGH
mitigation:
  immediate: "Fix VPC routing or security group rules"
  long_term: "Implement VPC connectivity monitoring and redundant NAT gateways"
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

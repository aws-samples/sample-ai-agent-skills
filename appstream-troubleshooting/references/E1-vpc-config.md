---
title: "E1 — VPC Configuration Issues"
description: "Diagnose AppStream 2.0 VPC configuration problems"
status: active
severity: HIGH
triggers:
  - "VPC configuration error"
  - "fleet VPC issue"
  - "subnet configuration"
  - "security group AppStream"
owner: devops-agent
objective: "Identify and resolve VPC configuration issues for AppStream fleets"
context: "AppStream fleets can run in a VPC for access to internal resources. VPC configuration requires at least two subnets in different AZs, security groups, and proper routing. Without a VPC, fleets use AppStream-managed networking."
---

## Phase 1 — Triage

MUST:
- Check fleet VPC configuration: `aws appstream describe-fleets --names <fleet-name> --query 'Fleets[*].VpcConfig'`
- Verify subnets exist and are in different AZs: `aws ec2 describe-subnets --subnet-ids <subnet-ids>`
- Check security groups: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Verify subnets have available IP addresses

SHOULD:
- Check route tables for the subnets: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify NAT gateway or VPC endpoints for AWS service access
- Check NACLs on the subnets

MAY:
- Test connectivity from a test instance in the same subnets
- Check VPC Flow Logs for blocked traffic

## Phase 2 — Remediate

MUST:
- Use at least two subnets in different AZs
- Configure security groups to allow required traffic (outbound HTTPS 443)
- Ensure subnets have sufficient IP addresses

SHOULD:
- Set up NAT gateway for internet access from private subnets
- Create VPC endpoints for AWS services (S3, AppStream)
- Allow outbound traffic to AppStream service endpoints

MAY:
- Use dedicated subnets for AppStream fleets
- Document VPC architecture for AppStream

## Common Issues

- symptoms: "Fleet fails to start in VPC"
  diagnosis: "Subnets in same AZ, insufficient IPs, or security group blocking."
  resolution: "Use subnets in different AZs. Ensure sufficient IPs. Allow outbound HTTPS."

- symptoms: "Fleet running but cannot access internal resources"
  diagnosis: "Security groups or route tables blocking internal traffic."
  resolution: "Allow traffic to internal resources in security groups and route tables."

- symptoms: "Fleet cannot reach AWS services"
  diagnosis: "No NAT gateway or VPC endpoints for AWS service access."
  resolution: "Add NAT gateway or create VPC endpoints for required AWS services."

## Output Format

```yaml
root_cause: "vpc_configuration_issue — <specific_cause>"
evidence:
  - type: vpc_config
    content: "<fleet VPC configuration>"
  - type: subnet_status
    content: "<subnet AZs and available IPs>"
  - type: security_groups
    content: "<security group rules>"
severity: HIGH
mitigation:
  immediate: "Fix VPC configuration"
  long_term: "Automate VPC setup with IaC"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Use at least two subnets in different AZs | GREEN | Best practice |
| Configure security groups | YELLOW | Security group rule change |
| Ensure sufficient IP addresses | GREEN | Capacity check |
| Set up NAT gateway | YELLOW | Network infrastructure change |
| Create VPC endpoints | YELLOW | New VPC endpoint creation |
| Allow outbound to AppStream endpoints | YELLOW | Security group change |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **VPC configuration**: Subnet, security group, route table details — INTERNAL
- **Network topology**: Fleet networking architecture — INTERNAL
- **Security group rules**: Inbound/outbound access rules — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER modify security groups on a fleet with active sessions without testing
- NEVER remove NAT gateway without verifying fleet connectivity

## Phase 3 — Rollback

1. Revert security group rule changes
2. Revert route table changes
3. Restore NAT gateway if it was removed
4. Verify fleet instances can communicate with AppStream service
5. Monitor fleet health after VPC changes

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

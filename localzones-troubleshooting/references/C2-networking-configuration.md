---
title: "C2 — Networking Configuration"
description: "Diagnose networking configuration issues in Local Zones"
status: active
severity: HIGH
triggers: ["networking error", "no internet access", "cannot reach parent region", "route table issue", "DNS not resolving"]
owner: devops-agent
objective: "Resolve networking configuration issues for Local Zone resources"
context: "Local Zone subnets share the VPC internet gateway with the parent region. Route tables, security groups, and NACLs must be properly configured. Traffic between Local Zone and parent region traverses the AWS backbone."
---

## Phase 1 — Triage

MUST:
- Check route table for Local Zone subnet: `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<subnet-id>" --query 'RouteTables[0].Routes'`
- Verify internet gateway attachment: `aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=<vpc-id>"`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[0].{Ingress:IpPermissions,Egress:IpPermissionsEgress}'`

SHOULD:
- Verify network ACL rules for the Local Zone subnet
- Check DNS resolution settings for the VPC
- Test connectivity from instance: ping, traceroute, nslookup

MAY:
- Check VPC flow logs for dropped traffic
- Verify NAT gateway configuration if using private subnets

## Phase 2 — Remediate

MUST:
- Add route to internet gateway for public subnets: `aws ec2 create-route --route-table-id <rtb-id> --destination-cidr-block 0.0.0.0/0 --gateway-id <igw-id>`
- Fix security group rules to allow required traffic
- Ensure DNS resolution is enabled on VPC

SHOULD:
- Configure NAT gateway in parent region for private Local Zone subnets
- Set up proper network ACL rules allowing required traffic

MAY:
- Configure VPC endpoints for AWS service access without internet
- Set up Direct Connect for dedicated connectivity to Local Zone

## Safety Ratings

```
safety_ratings:
  - "describe-route-tables: GREEN — read-only routing check"
  - "describe-security-groups: GREEN — read-only security check"
  - "create-route: YELLOW — modifies network routing"
  - "authorize-security-group-ingress: YELLOW — opens network access"
  - "replace-network-acl-association: YELLOW — changes subnet-level firewall"
```

## Escalation Conditions

- Networking changes require VPC-level modifications affecting other subnets
- Direct Connect setup needed for Local Zone requiring network team
- Security group changes require security team approval
- NAT gateway in parent region adds latency unacceptable for the workload

## Data Sensitivity

- Route table configuration: MEDIUM — reveals network topology
- Security group rules: MEDIUM — reveals allowed traffic patterns
- Network ACL rules: MEDIUM — reveals subnet-level access controls
- VPC flow logs: HIGH — contain source/destination IPs and traffic patterns

## Prohibited Actions

- NEVER open security groups to 0.0.0.0/0 for SSH/RDP without justification
- NEVER modify route tables without understanding impact on all subnets using them
- NEVER delete the default route in a route table without adding a replacement
- NEVER disable VPC DNS resolution as it breaks many AWS service integrations

## Phase 3 — Rollback

1. If route was added incorrectly, delete it: `aws ec2 delete-route --route-table-id <rtb-id> --destination-cidr-block <cidr>`
2. If security group was opened too broadly, revoke rules: `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol <proto> --port <port> --cidr <cidr>`
3. If network ACL was modified, restore previous rules
4. Test connectivity after rollback to verify network access is restored

## Common Issues

- symptoms: "Instance in Local Zone cannot reach the internet"
  diagnosis: "Route table missing default route to internet gateway or subnet is private without NAT."
  resolution: "Add 0.0.0.0/0 route to IGW for public subnets. Configure NAT gateway for private subnets."

- symptoms: "Instance in Local Zone cannot reach parent region resources"
  diagnosis: "Route table or security group blocking intra-VPC traffic."
  resolution: "Verify local route exists in route table. Check security groups allow traffic from Local Zone CIDR."

## Output Format

```yaml
root_cause: "networking — <specific_cause>"
evidence:
  - type: network_config
    content: "<route table, security group, and NACL details>"
severity: HIGH
mitigation:
  immediate: "Fix routing or security group rules"
  long_term: "Implement network configuration as IaC"
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

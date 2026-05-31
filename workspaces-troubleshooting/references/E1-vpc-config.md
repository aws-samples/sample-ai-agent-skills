---
title: "E1 — VPC Config"
description: "Diagnose VPC configuration issues affecting WorkSpaces"
status: active
severity: HIGH
triggers:
  - "VPC configuration error"
  - "subnet issues"
  - "security group blocking"
  - "NACL blocking workspace"
owner: devops-agent
objective: "Resolve VPC networking configuration issues for WorkSpaces"
context: "WorkSpaces require properly configured VPC with two subnets in different AZs, appropriate security groups, NACLs, and route tables. Security groups must allow streaming protocol traffic and directory communication."
---

## Phase 1 — Triage

MUST:
- Check registered subnets: `aws workspaces describe-workspace-directories --query 'Directories[].{DirId:DirectoryId,Subnets:SubnetIds}'`
- Verify subnet configuration: `aws ec2 describe-subnets --subnet-ids <subnet-1> <subnet-2> --query 'Subnets[].{SubnetId:SubnetId,VpcId:VpcId,AZ:AvailabilityZone,CIDR:CidrBlock,AvailableIPs:AvailableIpAddressCount}'`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Verify NACLs: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`

SHOULD:
- Check route tables for proper routing: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify DHCP options set: `aws ec2 describe-dhcp-options --dhcp-options-ids <dhcp-id>`
- Check VPC DNS settings: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`

MAY:
- Review VPC flow logs for dropped traffic
- Check for overlapping CIDR ranges with on-premises networks

## Phase 2 — Remediate

MUST:
- Ensure subnets are in different AZs with available IPs
- Configure security groups to allow ports 4172/4195 (streaming), 443 (auth), and directory ports
- Fix NACL rules if blocking required traffic

SHOULD:
- Set DHCP options with directory DNS servers
- Enable DNS resolution and DNS hostnames on the VPC
- Ensure route tables have appropriate routes

MAY:
- Create dedicated subnets for WorkSpaces to isolate traffic
- Implement VPC flow logs for ongoing monitoring

## Common Issues

- symptoms: "WorkSpaces cannot communicate with directory"
  diagnosis: "Security group missing rules for directory ports (53, 88, 389, 445)."
  resolution: "Add inbound/outbound rules for directory communication ports."

- symptoms: "Subnets running out of IP addresses"
  diagnosis: "Subnet CIDR too small for the number of WorkSpaces."
  resolution: "Use larger subnets or register additional subnets with the directory."

- symptoms: "WorkSpaces in private subnet cannot reach AWS services"
  diagnosis: "Missing NAT gateway or VPC endpoints for AWS services."
  resolution: "Add NAT gateway or create VPC endpoints for required services."

## Output Format

```yaml
root_cause: "vpc_config — <specific_cause>"
evidence:
  - type: subnet_config
    content: "<subnet details and available IPs>"
  - type: security_rules
    content: "<security group and NACL rules>"
severity: HIGH
mitigation:
  immediate: "Fix security groups, NACLs, or subnet configuration"
  long_term: "Plan subnet sizing and implement VPC flow log monitoring"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- WorkSpaces contain user desktop data and files
- Directory integration reveals Active Directory structure
- Connection logs contain user access patterns
- Classification: CONFIDENTIAL - user workspace data and directory information

## Prohibited Actions

- NEVER terminate or rebuild WorkSpaces without user data backup confirmation
- NEVER modify directory service configurations during business hours
- NEVER change MFA settings without user notification and transition period
- NEVER modify VPC or subnet configurations for running WorkSpaces without maintenance window

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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

---
title: "E1 — VPC Configuration Issues"
description: "Diagnose VPC networking issues affecting Batch jobs"
status: active
severity: HIGH
triggers:
  - "VPC configuration"
  - "subnet issue"
  - "network error"
  - "no internet access"
  - "DNS resolution failed"
owner: devops-agent
objective: "Identify and resolve VPC networking issues"
context: "VPC issues include: subnets without internet access (no NAT Gateway), security groups blocking outbound, DNS resolution failures, or insufficient IP addresses in subnets."
---

## Phase 1 — Triage

MUST:
- Check CE subnet configuration: `aws batch describe-compute-environments --compute-environments <ce-name> --query 'computeEnvironments[*].computeResources.{Subnets:subnets,SecurityGroups:securityGroupIds}'`
- Check subnet route tables for internet access: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify NAT Gateway exists for private subnets
- Check security group outbound rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[*].IpPermissionsEgress'`

SHOULD:
- Verify subnets have available IP addresses
- Check DNS resolution in VPC
- Verify VPC endpoints if using private subnets without NAT

MAY:
- Check NACLs for port blocking
- Verify VPC Flow Logs for blocked traffic

## Phase 2 — Remediate

MUST:
- Ensure subnets have route to internet (NAT Gateway for private, IGW for public)
- Allow outbound HTTPS (443) in security groups for ECR/S3 access
- Ensure DNS resolution works in VPC

SHOULD:
- Use VPC endpoints for ECR, S3, and CloudWatch Logs to avoid NAT costs
- Use private subnets with NAT Gateway for security

MAY:
- Implement VPC Flow Logs for network debugging
- Use multiple subnets across AZs for availability

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure subnets have route to internet | YELLOW | Routing change — affects all resources in subnet |
| Allow outbound HTTPS in security groups | YELLOW | Network change — verify no unintended access |
| Use VPC endpoints for ECR, S3, CloudWatch | GREEN | Adds network path — non-destructive |
| Use private subnets with NAT Gateway | GREEN | Security best practice |
| Implement VPC Flow Logs | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Compute environment runs production batch jobs
- VPC changes affect other services in the same network
- Network configuration is managed by a separate networking team
- NAT Gateway costs are a concern for high-throughput workloads

## Data Sensitivity

- **Classification: MEDIUM**
- Subnet and routing configuration reveals network architecture
- Security group rules expose network access boundaries
- VPC endpoint configuration reveals service access patterns
- DNS settings may reveal internal service discovery architecture

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest opening all ports in security groups to resolve connectivity
- **NEVER** suggest placing compute environments in public subnets without security justification
- **NEVER** suggest removing NACLs entirely to resolve network issues

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Modified route tables | Revert route table entries to previous configuration |
| Changed security group rules | Revert security group rules to previous state |
| Created VPC endpoints | Delete VPC endpoints if not needed |

## Output Format

```yaml
root_cause: "vpc_config_issue — <specific_cause>"
evidence:
  - type: subnet_config
    content: "<subnet and routing>"
  - type: security_groups
    content: "<outbound rules>"
severity: HIGH
mitigation:
  immediate: "Fix routing, security groups, or DNS"
  long_term: "Implement VPC endpoints and proper network architecture"
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
  - "NEVER suggest overly broad compute environment permissions"
  - "NEVER suggest disabling job queue scheduling"
  - "NEVER suggest public subnet placement for compute environments"

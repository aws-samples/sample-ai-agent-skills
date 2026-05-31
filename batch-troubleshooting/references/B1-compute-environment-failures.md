---
title: "B1 — Compute Environment Failures"
description: "Diagnose compute environment creation and scaling failures"
status: active
severity: HIGH
triggers:
  - "compute environment INVALID"
  - "CE creation failed"
  - "CE not scaling"
  - "compute environment error"
  - "CE disabled"
owner: devops-agent
objective: "Identify and resolve compute environment failures"
context: "CE failures include: INVALID state due to IAM/VPC issues, scaling failures, instance launch errors, or configuration problems."
---

## Phase 1 — Triage

MUST:
- Check CE status: `aws batch describe-compute-environments --compute-environments <ce-name> --query 'computeEnvironments[*].{State:state,Status:status,StatusReason:statusReason}'`
- Check CE configuration: `aws batch describe-compute-environments --compute-environments <ce-name> --query 'computeEnvironments[*].computeResources'`
- Verify IAM service role: check the service role has AWSBatchServiceRole permissions
- Verify instance role/profile exists and has ECS permissions

SHOULD:
- Check VPC subnets exist and have IPs: `aws ec2 describe-subnets --subnet-ids <subnet-ids>`
- Verify security groups exist: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Check instance type availability in configured AZs
- Review CloudTrail for CE update events

MAY:
- Check ECS cluster associated with CE
- Verify launch template if configured

## Phase 2 — Remediate

MUST:
- Fix INVALID CE by updating configuration: `aws batch update-compute-environment --compute-environment <ce-name> --compute-resources subnets=<subnets>,securityGroupIds=<sgs>`
- Fix IAM roles and instance profiles
- Ensure subnets and security groups are valid

SHOULD:
- Use multiple subnets across AZs for availability
- Add instance type diversity
- Set appropriate min/max/desired vCPUs

MAY:
- Recreate CE if update doesn't fix INVALID state
- Use managed CE for automatic infrastructure management

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix INVALID CE by updating configuration | YELLOW | CE update — may cause brief scaling pause |
| Fix IAM roles and instance profiles | YELLOW | Permission change — review scope |
| Ensure subnets and security groups are valid | GREEN | Verification — read-only |
| Recreate CE if update doesn't fix INVALID | RED | Deletes and recreates — migrate job queues |
| Use managed CE for automatic management | GREEN | Best practice recommendation |

## Escalation Conditions

- Compute environment runs production batch jobs
- CE in INVALID state blocking all job execution
- CE recreation requires job queue reassociation
- IAM role changes affect other services

## Data Sensitivity

- **Classification: MEDIUM**
- CE configuration reveals infrastructure architecture and instance types
- IAM role details expose service permissions
- VPC/subnet configuration reveals network architecture
- ECS cluster association reveals container orchestration setup

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest deleting a CE without first disassociating it from job queues
- **NEVER** suggest using overly permissive IAM roles to resolve CE failures
- **NEVER** suggest disabling CE without a replacement for queued jobs

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Updated CE configuration | Revert CE configuration to previous settings |
| Fixed IAM roles | Revert IAM role changes if they cause other issues |
| Recreated CE | Cannot undo — reassociate job queues with new CE |

## Output Format

```yaml
root_cause: "ce_failure — <specific_cause>"
evidence:
  - type: ce_status
    content: "<CE state and status reason>"
  - type: configuration
    content: "<CE configuration>"
severity: HIGH
mitigation:
  immediate: "Fix CE configuration or IAM roles"
  long_term: "Implement CE monitoring and use IaC"
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

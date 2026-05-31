---
title: "B2 — Runtime Environment Issues"
description: "Diagnose runtime environment creation and configuration issues"
status: active
severity: HIGH
triggers: ["environment creation failed", "environment not available", "environment stuck creating", "runtime error"]
owner: devops-agent
objective: "Resolve runtime environment creation and configuration issues"
context: "Runtime environments provide the compute infrastructure for M2 applications. They run in VPCs and require correct networking, instance types, and engine configuration."
---

## Phase 1 — Triage

MUST:
- Check environment status: `aws m2 get-environment --environment-id <env-id>`
- List all environments: `aws m2 list-environments --query 'environments[*].{Name:name,Id:environmentId,Status:status,EngineType:engineType}'`
- Review CloudTrail for environment events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=m2.amazonaws.com --max-results 20`

SHOULD:
- Verify VPC, subnet, and security group configuration
- Check IAM service role permissions
- Verify instance type availability in the selected subnets

MAY:
- Check CloudWatch metrics for environment health
- Review VPC flow logs for network issues

## Phase 2 — Remediate

MUST:
- Ensure VPC subnets have sufficient IP addresses
- Verify security groups allow required traffic (inbound/outbound)
- Fix IAM service role with required M2, EC2, and S3 permissions

SHOULD:
- Use private subnets with NAT gateway for internet access
- Configure appropriate instance types for the workload

MAY:
- Implement environment creation via IaC for consistency
- Set up CloudWatch alarms for environment health

## Common Issues

- symptoms: "Environment stuck in Creating status for more than 30 minutes"
  diagnosis: "VPC networking issue or IAM role permission problem."
  resolution: "Check subnet IP availability. Verify security groups. Ensure IAM role has ec2:CreateNetworkInterface and required M2 permissions."

## Safety Ratings

```
safety_ratings:
  - "get-environment: GREEN — read-only environment check"
  - "list-environments: GREEN — read-only enumeration"
  - "create-environment: YELLOW — creates compute infrastructure with cost implications"
  - "delete-environment: RED — destroys runtime environment and stops all applications"
```

## Escalation Conditions

- Environment creation fails repeatedly with internal errors
- Environment enters Failed state with no clear error message
- VPC networking issues require infrastructure team involvement
- Instance type not available in the selected AZ

## Data Sensitivity

- Environment configuration: MEDIUM — reveals infrastructure architecture
- VPC/subnet details: MEDIUM — reveals network topology
- IAM role policies: MEDIUM — reveals service permissions
- CloudTrail events: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER delete a runtime environment with running applications without stopping them first
- NEVER modify VPC security groups without understanding the impact on running applications
- NEVER create environments in public subnets for production workloads
- NEVER use overly permissive security groups (0.0.0.0/0 inbound) for M2 environments

## Phase 3 — Rollback

1. If environment creation failed, delete the failed environment: `aws m2 delete-environment --environment-id <env-id>`
2. Fix the root cause (networking, IAM, etc.) before recreating
3. If an existing environment was modified, restore previous configuration
4. Verify applications can be deployed after environment restoration
5. Document the environment issue and resolution

## Output Format

```yaml
root_cause: "runtime_environment — <specific_cause>"
evidence:
  - type: environment_status
    content: "<environment details and error information>"
severity: HIGH
mitigation:
  immediate: "Fix environment issue"
  long_term: "Implement environment creation via IaC with validation"
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

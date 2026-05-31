---
title: "D1 — IAM Role Configuration"
description: "Diagnose IAM role configuration issues for ECS Anywhere"
status: active
severity: HIGH
triggers: ["IAM error", "access denied", "role not found", "permission denied", "task role error"]
owner: devops-agent
objective: "Resolve IAM role configuration issues for ECS Anywhere"
context: "ECS Anywhere uses multiple IAM roles: SSM activation role, task execution role, and task role. Each has different requirements."
---

## Phase 1 — Triage

MUST:
- Check task execution role: `aws ecs describe-task-definition --task-definition <task-def> --query 'taskDefinition.executionRoleArn'`
- Check task role: `aws ecs describe-task-definition --task-definition <task-def> --query 'taskDefinition.taskRoleArn'`
- Verify the SSM activation role: `aws iam get-role --role-name <ecs-anywhere-role> --query 'Role.AssumeRolePolicyDocument'`

SHOULD:
- Verify task execution role has ECR pull and CloudWatch Logs permissions
- Check task role has permissions required by the application
- Verify the SSM activation role trusts ssm.amazonaws.com

MAY:
- Use IAM Policy Simulator to test permissions
- Check for SCPs that may restrict ECS Anywhere roles
- Review CloudTrail for AccessDenied events

## Phase 2 — Remediate

MUST:
- Ensure SSM activation role trusts ssm.amazonaws.com and has required SSM permissions
- Ensure task execution role has ecr:GetAuthorizationToken, ecr:BatchGetImage, logs:CreateLogStream, logs:PutLogEvents
- Ensure task role has application-specific permissions

SHOULD:
- Use AWS managed policies where appropriate (AmazonSSMManagedInstanceCore, AmazonECSTaskExecutionRolePolicy)
- Scope task role permissions to least privilege
- Verify role trust policies include the correct service principals

MAY:
- Implement role session tagging for audit trails
- Set up IAM Access Analyzer to review role permissions

## Common Issues

- symptoms: "Task fails with 'unable to pull secrets or registry auth'"
  diagnosis: "Task execution role missing ECR or Secrets Manager permissions."
  resolution: "Attach AmazonECSTaskExecutionRolePolicy to the execution role. Add secretsmanager:GetSecretValue if using secrets."

- symptoms: "Application in container gets AccessDenied calling AWS APIs"
  diagnosis: "Task role missing required permissions, or task role not configured."
  resolution: "Add required permissions to the task role. Ensure taskRoleArn is set in the task definition."

## Safety Ratings

```
safety_ratings:
  - "ecs describe-task-definition: GREEN — read-only"
  - "iam get-role: GREEN — read-only"
  - "iam get-role-policy: GREEN — read-only"
  - "iam attach-role-policy: RED — modifies IAM permissions"
  - "iam simulate-principal-policy: GREEN — read-only simulation"
```

## Escalation Conditions

- IAM changes require security team approval
- SCPs block required permissions at the organization level
- Cross-account role assumptions needed for the application
- Role permissions conflict with compliance requirements

## Data Sensitivity

- `aws ecs describe-task-definition`: MEDIUM — reveals role ARNs and configuration
- `aws iam get-role`: MEDIUM — reveals trust policy
- `aws iam get-role-policy`: HIGH — reveals permission details
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER grant AdministratorAccess to ECS task roles
- NEVER use the SSM activation role as the task role
- NEVER embed AWS credentials in container environment variables
- NEVER modify shared IAM roles without impact analysis on other services

## Phase 3 — Rollback

1. If IAM policies were modified, revert to previous policy versions
2. If role trust policies were changed, restore previous trust relationships
3. If task definition was updated with wrong roles, revert to previous version
4. Verify tasks can start and access required resources after rollback
5. Document the IAM configuration requirements

## Output Format

```yaml
root_cause: "iam_configuration — <specific_cause>"
evidence:
  - type: iam_error
    content: "<role and permission details>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role configuration"
  long_term: "Implement IAM policy-as-code with automated validation"
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
  - command: "describe-tasks"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-services"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "execute-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest running containers as root to fix permission issues"
  - "NEVER suggest disabling task networking"
  - "NEVER remove IAM task role to troubleshoot"

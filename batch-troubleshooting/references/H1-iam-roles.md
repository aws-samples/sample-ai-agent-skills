---
title: "H1 — IAM Role Issues"
description: "Diagnose IAM role and permission problems in Batch"
status: active
severity: HIGH
triggers:
  - "IAM permission denied"
  - "role error"
  - "execution role"
  - "instance role"
  - "service role"
owner: devops-agent
objective: "Identify and resolve IAM role issues"
context: "Batch uses multiple IAM roles: service role (Batch service), execution role (ECS task execution for Fargate), job role (container permissions), and instance role (EC2 instances). Each has different requirements."
---

## Phase 1 — Triage

MUST:
- Check CE service role: `aws batch describe-compute-environments --compute-environments <ce-name> --query 'computeEnvironments[*].serviceRole'`
- Check job definition execution role: `aws batch describe-job-definitions --job-definitions <def-name> --query 'jobDefinitions[*].containerProperties.executionRoleArn'`
- Check job role: `aws batch describe-job-definitions --job-definitions <def-name> --query 'jobDefinitions[*].containerProperties.jobRoleArn'`
- Verify role policies: `aws iam list-attached-role-policies --role-name <role-name>`

SHOULD:
- Verify service-linked role exists: AWSServiceRoleForBatch
- Check execution role has ECR and CloudWatch Logs permissions
- Verify job role has permissions needed by the application
- Check instance profile for EC2 CE

MAY:
- Check for SCP restrictions
- Review CloudTrail for access denied events

## Phase 2 — Remediate

MUST:
- Execution role needs: ecr:GetAuthorizationToken, ecr:BatchGetImage, ecr:GetDownloadUrlForLayer, logs:CreateLogStream, logs:PutLogEvents
- Job role needs: application-specific permissions (S3, DynamoDB, etc.)
- Instance role needs: ECS container instance permissions

SHOULD:
- Use AWS managed policies where appropriate
- Implement least-privilege for job roles

MAY:
- Use IAM Access Analyzer to right-size permissions
- Implement role session tags for fine-grained access

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Add ECR and CloudWatch permissions to execution role | YELLOW | Permission change — review scope |
| Add application-specific permissions to job role | YELLOW | Permission change — review scope |
| Add ECS permissions to instance role | YELLOW | Permission change — review scope |
| Use AWS managed policies | GREEN | Standard policies — well-scoped |
| Implement least-privilege for job roles | GREEN | Security best practice |

## Escalation Conditions

- Compute environment runs production batch jobs
- IAM role changes affect other services sharing the same roles
- Permission errors block all job executions
- Role creation requires organizational IAM approval process

## Data Sensitivity

- **Classification: HIGH**
- IAM role policies reveal infrastructure access patterns
- Execution role permissions expose container registry and logging access
- Job role permissions reveal application data access boundaries
- Instance role permissions expose EC2-level access controls

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest granting wildcard (*) permissions to any Batch role
- **NEVER** suggest using the same role for execution, job, and instance purposes
- **NEVER** suggest deleting IAM roles while jobs are running

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Added permissions to execution role | Remove added permissions if not needed |
| Added permissions to job role | Remove added permissions after verifying impact |
| Modified instance role | Revert instance role policy to previous version |

## Output Format

```yaml
root_cause: "iam_role_issue — <specific_cause>"
evidence:
  - type: role_config
    content: "<role ARN and policies>"
  - type: permission_error
    content: "<specific denied action>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role permissions"
  long_term: "Implement least-privilege roles with monitoring"
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

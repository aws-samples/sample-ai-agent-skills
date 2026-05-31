---
title: "G1 — MWAA IAM Execution Role Issues"
description: "Diagnose IAM execution role permission problems in MWAA"
status: active
severity: HIGH
triggers:
  - "access denied"
  - "execution role"
  - "permission denied"
  - "IAM error"
  - "not authorized"
owner: devops-agent
objective: "Identify and resolve IAM execution role permission issues affecting MWAA operations"
context: "The MWAA execution role is the single IAM identity for all operations: S3 DAG access, CloudWatch logging, Secrets Manager, KMS, and any AWS service calls from DAGs. Permission errors are the most common MWAA issue."
---

## Phase 1 — Triage

MUST:
- Get execution role ARN: `aws mwaa get-environment --name <env-name> --query 'Environment.ExecutionRoleArn'`
- Check role trust policy: `aws iam get-role --role-name <role-name> --query 'Role.AssumeRolePolicyDocument'`
- List attached policies: `aws iam list-attached-role-policies --role-name <role-name>`
- List inline policies: `aws iam list-role-policies --role-name <role-name>`
- Check specific policy content: `aws iam get-policy-version --policy-arn <policy-arn> --version-id <version>`

SHOULD:
- Verify S3 permissions for DAG bucket: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:GetObject --resource-arns arn:aws:s3:::<bucket>/<dags-path>/*`
- Verify CloudWatch Logs permissions
- Check for deny policies or permission boundaries: `aws iam get-role --role-name <role-name> --query 'Role.PermissionsBoundary'`

MAY:
- Check SCP restrictions in AWS Organizations
- Verify KMS key policy allows the execution role

## Phase 2 — Remediate

MUST:
- Ensure trust policy includes `airflow.amazonaws.com` and `airflow-env.amazonaws.com`
- Grant S3 read access to DAG bucket, plugins, and requirements paths
- Grant CloudWatch Logs write permissions
- Grant SQS permissions for the MWAA-managed queue

SHOULD:
- Add permissions for any AWS services called from DAGs
- Use least-privilege policies scoped to specific resources
- Grant Secrets Manager read access if using secrets backend

MAY:
- Use AWS managed policy `AmazonMWAAFullConsoleAccess` as a reference
- Add KMS permissions if using customer-managed keys

## Common Issues

- symptoms: "Tasks fail with AccessDeniedException for S3/DynamoDB/etc."
  diagnosis: "Execution role lacks permissions for the AWS service the task calls."
  resolution: "Add the required service permissions to the execution role."

- symptoms: "Environment creation fails with role assumption error"
  diagnosis: "Trust policy missing MWAA service principals."
  resolution: "Add airflow.amazonaws.com and airflow-env.amazonaws.com to trust policy."

- symptoms: "DAGs not syncing from S3"
  diagnosis: "Execution role lacks s3:GetObject and s3:ListBucket for the DAG bucket."
  resolution: "Grant S3 read permissions scoped to the DAG bucket."

## Output Format

```yaml
root_cause: "iam_execution_role_issue — <specific_cause>"
evidence:
  - type: role_config
    content: "<trust policy and attached policies>"
  - type: permission_test
    content: "<simulate-principal-policy results>"
  - type: error_message
    content: "<specific access denied error>"
severity: HIGH
mitigation:
  immediate: "Add missing permissions to the execution role"
  long_term: "Implement least-privilege policies with regular access reviews"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — IAM role misconfigurations can grant excessive access
- Service Disruption Risk: HIGH — missing permissions can break environment functionality
- Compliance Impact: HIGH — IAM roles directly affect security and compliance posture

## Escalation Conditions
- Execution role permissions allow access to unauthorized resources
- Role trust policy modified to allow untrusted principals
- Permission boundary violations detected
- Execution role used by multiple environments with different security requirements
- AWS Organizations SCP blocks required MWAA permissions

## Data Sensitivity
- IAM role ARNs and policies — CONFIDENTIAL
- Trust policy configurations — CONFIDENTIAL
- Permission boundaries — CONFIDENTIAL
- Cross-account role assumptions — RESTRICTED
- Never share IAM role credentials or session tokens

## Prohibited Actions
- Do NOT attach AdministratorAccess or PowerUserAccess to MWAA execution roles
- Do NOT use wildcard (*) resource ARNs in execution role policies
- Do NOT share execution roles across environments with different security contexts
- Do NOT remove permission boundaries from execution roles
- Do NOT modify execution role trust policies without security review
- Do NOT grant S3:* or KMS:* without scoping to specific resources

## Phase 3 — Rollback
- Restore previous IAM policy version using policy versioning
- Revert trust policy changes to last known secure configuration
- Re-attach permission boundaries if removed
- Revoke any temporary credentials issued under the compromised policy
- Verify environment functionality after IAM policy rollback
- Conduct security review of all changes made to the execution role

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

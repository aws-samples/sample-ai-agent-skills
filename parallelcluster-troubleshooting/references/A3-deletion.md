---
title: "A3 — Cluster Deletion Issues"
description: "Diagnose ParallelCluster deletion failures"
status: active
severity: MEDIUM
triggers:
  - "cluster deletion failed"
  - "DELETE_FAILED"
  - "cannot delete cluster"
  - "stuck deleting"
owner: devops-agent
objective: "Resolve cluster deletion failures and clean up resources"
context: "Cluster deletion removes the CloudFormation stack and all associated resources. Failures may be due to resources in use, S3 bucket not empty, or IAM permission issues."
---

## Phase 1 — Triage

MUST:
- Check cluster status: `pcluster describe-cluster --cluster-name <cluster-name>`
- Check stack events: `aws cloudformation describe-stack-events --stack-name parallelcluster-<cluster-name> --query 'StackEvents[?ResourceStatus==`DELETE_FAILED`]'`
- Identify resources blocking deletion
- Check for running instances: `aws ec2 describe-instances --filters "Name=tag:parallelcluster:cluster-name,Values=<cluster-name>" "Name=instance-state-name,Values=running"`

SHOULD:
- Check S3 bucket contents (non-empty buckets block deletion)
- Verify IAM permissions for deletion
- Check for ENIs or security groups in use by other resources

MAY:
- Review CloudFormation stack resources for dependencies
- Check for manual modifications to stack resources

## Phase 2 — Remediate

MUST:
- If S3 bucket not empty: empty the bucket first
- If resources in use: identify and remove external dependencies
- If IAM blocking: add required deletion permissions
- Retry deletion: `pcluster delete-cluster --cluster-name <cluster-name>`

SHOULD:
- Back up important data before deletion (shared storage, job outputs)
- Use --force flag if normal deletion fails
- Clean up orphaned resources after forced deletion

MAY:
- Manually delete the CloudFormation stack if pcluster CLI fails
- Check for stack drift that may cause deletion issues

## Common Issues

- symptoms: "DELETE_FAILED on S3 bucket resource"
  diagnosis: "S3 bucket contains objects and cannot be deleted."
  resolution: "Empty the bucket: aws s3 rm s3://<bucket> --recursive. Retry deletion."

- symptoms: "DELETE_FAILED on security group"
  diagnosis: "Security group is referenced by resources outside the stack."
  resolution: "Remove external references to the security group. Retry deletion."

## Output Format

```yaml
root_cause: "deletion_failure — <specific_cause>"
evidence:
  - type: stack_events
    content: "<DELETE_FAILED events>"
  - type: blocking_resources
    content: "<resources preventing deletion>"
severity: MEDIUM
mitigation:
  immediate: "Remove blockers and retry deletion"
  long_term: "Implement clean deletion procedures and data backup"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — deletion may leave orphaned data
- Service Disruption Risk: HIGH — deletion terminates all cluster resources
- Compliance Impact: MEDIUM — data retention requirements must be met before deletion

## Escalation Conditions
- Cluster deletion fails leaving orphaned resources
- Deletion blocked by resource dependencies
- Shared storage data not backed up before deletion
- CloudFormation stack deletion fails
- Running jobs exist on cluster being deleted

## Data Sensitivity
- Cluster data and job outputs — CONFIDENTIAL
- Shared storage contents — CONFIDENTIAL
- CloudFormation stack resources — INTERNAL
- IAM roles and policies — INTERNAL
- Never delete data without verifying backup availability

## Prohibited Actions
- Do NOT delete clusters with running jobs without approval
- Do NOT delete clusters without backing up shared storage data
- Do NOT force-delete CloudFormation stacks without understanding orphaned resources
- Do NOT delete clusters without verifying data retention compliance
- Do NOT ignore deletion failures
- Do NOT delete clusters that other teams depend on without coordination

## Phase 3 — Rollback
- Re-create cluster from saved configuration if deletion was premature
- Restore shared storage data from backups
- Clean up orphaned resources from failed deletion
- Restore IAM roles if needed for re-creation
- Verify all resources are properly cleaned up after successful deletion
- Document deletion process and any issues encountered

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

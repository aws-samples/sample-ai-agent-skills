---
title: "G1 — IAM Permission Issues"
description: "Diagnose IAM permission problems for ParallelCluster operations"
status: active
severity: HIGH
triggers:
  - "IAM parallelcluster"
  - "permission denied"
  - "access denied"
  - "unauthorized"
owner: devops-agent
objective: "Resolve IAM permission issues for cluster creation, management, and operations"
context: "ParallelCluster requires extensive IAM permissions for cluster creation (CloudFormation, EC2, EFS, FSx, S3, IAM, Lambda). The head node and compute nodes use instance profiles with specific permissions."
---

## Phase 1 — Triage

MUST:
- Check the error message for the specific permission denied
- Verify user/role permissions: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names cloudformation:CreateStack ec2:RunInstances`
- Check instance profile on head node: `aws ec2 describe-instances --instance-ids <head-node-id> --query 'Reservations[*].Instances[*].IamInstanceProfile'`
- Review CloudTrail for AccessDenied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=<failed-action> --max-results 5`

SHOULD:
- Check for SCPs restricting ParallelCluster operations
- Verify the ParallelCluster IAM policies are attached
- Check permission boundaries on the role

MAY:
- Use IAM Access Analyzer for policy validation
- Review custom IAM policies for missing actions

## Phase 2 — Remediate

MUST:
- Add missing permissions identified in the error or CloudTrail
- Use ParallelCluster-provided IAM policy templates where available
- Ensure instance profiles have required permissions for AWS API calls

SHOULD:
- Follow least-privilege principle with ParallelCluster-specific policies
- Use separate roles for cluster creation vs runtime operations
- Document required permissions for the team

MAY:
- Implement IAM policy versioning and review process
- Set up automated permission auditing

## Common Issues

- symptoms: "Cluster creation fails with AccessDenied on iam:CreateRole"
  diagnosis: "User lacks permission to create IAM roles for ParallelCluster."
  resolution: "Add iam:CreateRole, iam:AttachRolePolicy, iam:CreateInstanceProfile permissions."

- symptoms: "Head node cannot call AWS APIs"
  diagnosis: "Instance profile missing or lacks required permissions."
  resolution: "Verify instance profile is attached and has required service permissions."

## Output Format

```yaml
root_cause: "iam — <specific_cause>"
evidence:
  - type: error_message
    content: "<AccessDenied details>"
  - type: cloudtrail
    content: "<denied API call>"
severity: HIGH
mitigation:
  immediate: "Add missing IAM permissions"
  long_term: "Implement ParallelCluster-specific IAM policies"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — IAM controls all access to cluster resources
- Service Disruption Risk: MEDIUM — IAM issues prevent legitimate access
- Compliance Impact: HIGH — IAM is fundamental to security compliance

## Escalation Conditions
- IAM policy changes lock out cluster administrators
- Overly permissive IAM policies detected
- Cross-account access issues affect cluster operations
- Service-linked role issues prevent cluster management
- IAM audit reveals unauthorized access patterns

## Data Sensitivity
- IAM policies and role configurations — CONFIDENTIAL
- Access patterns and audit logs — CONFIDENTIAL
- Service-linked role configurations — INTERNAL
- Cross-account trust policies — CONFIDENTIAL
- Never expose IAM credentials or session tokens

## Prohibited Actions
- Do NOT attach AdministratorAccess to cluster roles
- Do NOT use wildcard resource ARNs in cluster IAM policies
- Do NOT share IAM credentials across teams
- Do NOT bypass IAM authentication for troubleshooting
- Do NOT ignore IAM access audit findings
- Do NOT modify service-linked roles without understanding impact

## Phase 3 — Rollback
- Restore previous IAM policy versions
- Revert trust policy changes
- Re-enable permission boundaries if removed
- Revoke compromised credentials
- Verify access controls after IAM changes
- Document IAM changes and conduct security review

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

---
title: "F2 — IAM Permissions"
description: "Diagnose Managed Flink IAM permission issues"
status: active
severity: HIGH
triggers: ["access denied", "IAM error", "permission denied", "role error", "unauthorized"]
owner: devops-agent
objective: "Resolve IAM permission issues including application role, source/sink access, and service-linked role"
context: "Managed Flink applications use an IAM role for accessing sources, sinks, S3 code, CloudWatch Logs, and VPC resources."
---

## Phase 1 — Triage

MUST:
- Check application IAM role: `aws kinesisanalyticsv2 describe-application --application-name <app-name> --query 'ApplicationDetail.ServiceExecutionRole'`
- Verify role trust policy: `aws iam get-role --role-name <role-name> --query 'Role.AssumeRolePolicyDocument'`
- Check role permissions: `aws iam list-attached-role-policies --role-name <role-name>`

SHOULD:
- Review inline policies: `aws iam list-role-policies --role-name <role-name>`
- Test specific permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names kinesis:GetRecords kinesis:DescribeStream`

MAY:
- Review CloudTrail for access denied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=GetRecords --max-results 10`

## Phase 2 — Remediate

MUST:
- Ensure role trusts kinesisanalytics.amazonaws.com in trust policy
- Grant permissions for source access (kinesis:GetRecords, kinesis:GetShardIterator, kinesis:DescribeStream, kinesis:ListShards)
- Grant permissions for sink access (kinesis:PutRecord, kinesis:PutRecords, s3:PutObject)

SHOULD:
- Grant CloudWatch Logs permissions (logs:PutLogEvents, logs:DescribeLogGroups, logs:DescribeLogStreams)
- Grant S3 permissions for code artifact access (s3:GetObject)

MAY:
- Grant VPC permissions if VPC-connected (ec2:DescribeVpcs, ec2:DescribeSubnets, ec2:DescribeSecurityGroups, ec2:DescribeNetworkInterfaces, ec2:CreateNetworkInterface, ec2:DeleteNetworkInterface)

## Safety Ratings

```
safety_ratings:
  - "describe-application: GREEN — read-only application details"
  - "iam get-role: GREEN — read-only role details"
  - "iam simulate-principal-policy: GREEN — read-only permission test"
  - "iam attach-role-policy: RED — modifies IAM permissions"
```

## Escalation Conditions

- IAM role changes require security team approval
- Cross-account access requires role assumption configuration
- Permission boundary restricts required permissions
- SCP blocks required API calls

## Data Sensitivity

- `aws iam get-role`: MEDIUM — reveals role configuration and trust relationships
- `aws iam list-attached-role-policies`: MEDIUM — reveals permission scope
- `aws iam simulate-principal-policy`: LOW — tests permissions without making changes

## Prohibited Actions

- NEVER grant * permissions on any resource as a troubleshooting shortcut
- NEVER modify IAM roles used by other applications without coordination
- NEVER remove permissions without understanding which operations they enable
- NEVER use the account root user for Managed Flink application roles

## Phase 3 — Rollback

1. If IAM role permissions were changed, revert to previous policy
2. If trust policy was modified, restore original trust relationships
3. Verify application still functions with reverted permissions
4. Check other services using the same role are not affected
5. Document the minimum required permissions for the application

## Common Issues

- symptoms: "Application logs show 'User: arn:aws:sts::...:assumed-role/... is not authorized to perform: kinesis:GetRecords'"
  diagnosis: "Application IAM role does not have kinesis:GetRecords permission on the source stream."
  resolution: "Add kinesis:GetRecords, kinesis:GetShardIterator, kinesis:DescribeStream, kinesis:ListShards to the role policy for the source stream ARN."

## Output Format

```yaml
root_cause: "iam_permissions — <specific_cause>"
evidence:
  - type: iam_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions"
  long_term: "Use least-privilege policies"
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

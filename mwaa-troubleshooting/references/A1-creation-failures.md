---
title: "A1 — MWAA Environment Creation Failures"
description: "Diagnose why an Amazon MWAA environment fails to create"
status: active
severity: HIGH
triggers:
  - "environment creation failed"
  - "create environment error"
  - "CREATE_FAILED"
  - "environment stuck creating"
owner: devops-agent
objective: "Identify why an MWAA environment fails to create and resolve the blocking issue"
context: "MWAA environment creation requires a valid S3 bucket, execution role, VPC with private subnets, NAT gateway, and security groups. Failures typically stem from IAM permission issues, VPC misconfiguration, or S3 bucket access problems."
---

## Phase 1 — Triage

MUST:
- Check environment status: `aws mwaa get-environment --name <env-name> --query 'Environment.{Status:Status,StatusReason:StatusReason}'`
- List environments: `aws mwaa list-environments`
- Check CloudTrail for creation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateEnvironment --max-results 10`
- Verify execution role exists: `aws iam get-role --role-name <execution-role-name>`
- Check execution role trust policy: `aws iam get-role --role-name <execution-role-name> --query 'Role.AssumeRolePolicyDocument'`

SHOULD:
- Verify S3 bucket exists and is accessible: `aws s3 ls s3://<bucket-name>/`
- Check S3 bucket policy: `aws s3api get-bucket-policy --bucket <bucket-name>`
- Verify VPC subnets are private: `aws ec2 describe-subnets --subnet-ids <subnet-ids> --query 'Subnets[*].{SubnetId:SubnetId,AZ:AvailabilityZone,MapPublicIp:MapPublicIpOnLaunch}'`
- Check NAT gateway exists: `aws ec2 describe-nat-gateways --filter Name=vpc-id,Values=<vpc-id> --query 'NatGateways[*].{Id:NatGatewayId,State:State,SubnetId:SubnetId}'`

MAY:
- Check service quotas: `aws service-quotas get-service-quota --service-code airflow --quota-code L-3A3D004D`
- Verify KMS key access if using custom encryption: `aws kms describe-key --key-id <key-id>`

## Phase 2 — Remediate

MUST:
- Fix execution role trust policy to include `airflow.amazonaws.com` and `airflow-env.amazonaws.com`
- Ensure execution role has S3, CloudWatch Logs, and SQS permissions
- Use private subnets in at least two AZs with NAT gateway

SHOULD:
- Ensure S3 bucket has versioning enabled (required for MWAA)
- Verify security groups allow self-referencing inbound rules for MWAA components
- Ensure S3 bucket is in the same region as the MWAA environment

MAY:
- Retry creation after fixing identified issues
- Use CloudFormation or Terraform for repeatable provisioning

## Common Issues

- symptoms: "CREATE_FAILED with execution role error"
  diagnosis: "Execution role trust policy missing airflow.amazonaws.com or airflow-env.amazonaws.com."
  resolution: "Update trust policy to include both MWAA service principals."

- symptoms: "CREATE_FAILED with network error"
  diagnosis: "Subnets are public or NAT gateway is missing/inactive."
  resolution: "Use private subnets with active NAT gateway. Verify route tables point to NAT."

- symptoms: "CREATE_FAILED with S3 access error"
  diagnosis: "Execution role cannot access the S3 DAG bucket or bucket versioning is disabled."
  resolution: "Grant S3 permissions to execution role and enable bucket versioning."

## Output Format

```yaml
root_cause: "environment_creation_failure — <specific_cause>"
evidence:
  - type: environment_status
    content: "<status and statusReason>"
  - type: iam_role
    content: "<role trust policy and permissions>"
  - type: vpc_config
    content: "<subnet and NAT gateway details>"
severity: HIGH
mitigation:
  immediate: "Fix the identified IAM, VPC, or S3 issue and retry creation"
  long_term: "Use IaC templates with validated networking and IAM configurations"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — environment creation involves IAM roles and VPC configurations
- Service Disruption Risk: LOW — creation failures do not affect existing environments
- Compliance Impact: MEDIUM — misconfigured IAM roles may violate least-privilege principles

## Escalation Conditions
- Environment creation fails repeatedly after applying recommended fixes
- StatusReason indicates internal service error not related to customer configuration
- Creation failure involves cross-account IAM role assumptions
- Security team flags execution role permissions as overly broad
- Environment creation blocked by AWS Organizations SCP policies

## Data Sensitivity
- IAM role ARNs and trust policies — INTERNAL
- VPC subnet IDs and CIDR ranges — INTERNAL
- S3 bucket names and paths — INTERNAL
- KMS key IDs — CONFIDENTIAL
- CloudTrail event logs — INTERNAL
- Never expose AWS account IDs in external communications

## Prohibited Actions
- Do NOT create or attach IAM policies with `*` resource wildcards for MWAA execution roles
- Do NOT place MWAA environments in public subnets
- Do NOT disable S3 bucket versioning on DAG buckets
- Do NOT share MWAA execution role credentials or temporary tokens
- Do NOT modify VPC route tables without verifying NAT gateway connectivity
- Do NOT bypass KMS encryption requirements for MWAA environments

## Phase 3 — Rollback
- If creation fails, delete the failed environment: `aws mwaa delete-environment --name <env-name>`
- Remove any orphaned IAM roles created specifically for the failed environment
- Clean up security groups created for the MWAA environment
- Remove S3 bucket policies added for the failed environment if no longer needed
- Document the failure reason and remediation steps taken before retrying
- Verify no residual CloudWatch log groups remain from the failed creation

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

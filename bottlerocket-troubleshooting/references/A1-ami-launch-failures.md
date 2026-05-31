---
title: "A1 — AMI Launch Failures"
description: "Diagnose Bottlerocket AMI launch and instance start failures"
status: active
severity: HIGH
triggers: ["instance launch failed", "bottlerocket ami not found", "instance not starting", "launch template error"]
owner: devops-agent
objective: "Resolve Bottlerocket AMI launch failures and get instances running"
context: "Bottlerocket AMIs are region-specific, variant-specific (ECS/EKS), and architecture-specific (x86_64/arm64). AMI IDs change with each release."
---

## Phase 1 — Triage

MUST:
- Identify the target variant and architecture: `aws ssm get-parameter --name /aws/service/bottlerocket/aws-k8s-1.30/x86_64/latest/image_id --query 'Parameter.Value' --output text`
- Verify the AMI exists in the target region: `aws ec2 describe-images --image-ids <ami-id> --query 'Images[0].{State:State,Name:Name,Architecture:Architecture}'`
- Check instance launch errors: `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[0].Instances[0].{State:State.Name,StateReason:StateReason.Message}'`

SHOULD:
- Verify the instance type supports the AMI architecture (x86_64 vs arm64)
- Check subnet and security group configuration
- Verify IAM instance profile is attached: `aws ec2 describe-iam-instance-profile-associations --filters "Name=instance-id,Values=<instance-id>"`

MAY:
- Review launch template for user data format issues
- Check EC2 service quotas: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A`

## Phase 2 — Remediate

MUST:
- Use SSM parameter store to find the correct AMI: `aws ssm get-parameter --name /aws/service/bottlerocket/aws-k8s-1.30/x86_64/latest/image_id`
- Ensure user data is valid TOML format, not bash scripts
- Verify the instance type is compatible with the AMI architecture

SHOULD:
- Use launch templates with SSM parameter references for automatic AMI updates
- Ensure the subnet has internet access or VPC endpoints for SSM and ECR/EKS

MAY:
- Set up AMI update automation using EventBridge and Lambda

## Safety Ratings

```
safety_ratings:
  - "ssm get-parameter: GREEN — read-only lookup of AMI IDs"
  - "ec2 describe-images: GREEN — read-only AMI metadata query"
  - "ec2 describe-instances: GREEN — read-only instance status check"
  - "ec2 run-instances: YELLOW — launches new compute resources with cost implications"
```

## Escalation Conditions

- AMI not found in SSM parameter store for the required variant/version
- Instance repeatedly fails to start with InternalError state reasons
- Launch failures occur across multiple instance types and subnets
- GPU variant AMI required but not available in the target region

## Data Sensitivity

- `aws ssm get-parameter`: LOW — returns public AMI IDs only
- `aws ec2 describe-instances`: MEDIUM — may reveal VPC topology and security group details
- `aws ec2 describe-iam-instance-profile-associations`: MEDIUM — reveals IAM role assignments
- User data content: HIGH — may contain cluster certificates and API endpoints

## Prohibited Actions

- NEVER hardcode AMI IDs — always use SSM parameter store lookups
- NEVER launch instances with bash user data scripts — Bottlerocket requires TOML
- NEVER suggest using an ECS variant AMI for EKS clusters or vice versa
- NEVER launch instances without verifying the IAM instance profile has required permissions

## Phase 3 — Rollback

1. Terminate failed instances: `aws ec2 terminate-instances --instance-ids <instance-id>`
2. Revert launch template to previous working version: `aws ec2 modify-launch-template --launch-template-id <id> --default-version <previous-version>`
3. If auto scaling group was updated, roll back: `aws autoscaling update-auto-scaling-group --auto-scaling-group-name <name> --launch-template LaunchTemplateId=<id>,Version=<previous>`
4. Verify replacement instances launch successfully with the previous configuration
5. Document the AMI ID and variant that caused the failure

## Common Issues

- symptoms: "Instance launches but immediately enters shutting-down state"
  diagnosis: "User data is in bash format instead of TOML, or wrong AMI variant for the orchestrator."
  resolution: "Convert user data to TOML format and verify AMI variant matches the target orchestrator."

- symptoms: "AMI not found error during launch"
  diagnosis: "AMI ID is from a different region or has been deprecated."
  resolution: "Use SSM parameter store to get the current AMI ID for the target region."

## Output Format

```yaml
root_cause: "ami_launch — <specific_cause>"
evidence:
  - type: launch_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix AMI selection or user data format"
  long_term: "Use SSM parameter references in launch templates"
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

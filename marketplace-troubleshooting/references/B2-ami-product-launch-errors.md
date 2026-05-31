---
title: "B2 — AMI Product Launch Errors"
description: "Diagnose AMI-based Marketplace product launch failures"
status: active
severity: HIGH
triggers: ["ami launch failed", "cannot launch marketplace ami", "ec2 marketplace error", "ami product error"]
owner: devops-agent
objective: "Resolve AMI product launch failures from Marketplace"
context: "AMI products launch EC2 instances with pre-configured software. Launch failures can stem from subscription issues, instance type compatibility, or region availability."
---

## Phase 1 — Triage

MUST:
- Verify active subscription: `aws marketplace-entitlement get-entitlements --product-code <product-code>`
- Check EC2 launch errors: `aws ec2 describe-instances --filters "Name=image-id,Values=<ami-id>" --query 'Reservations[*].Instances[*].{State:State.Name,StateReason:StateReason}'`
- Verify AMI availability in the target region: `aws ec2 describe-images --image-ids <ami-id>`

SHOULD:
- Check instance type compatibility with the AMI
- Verify VPC, subnet, and security group configuration
- Review EC2 service quotas: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A`

MAY:
- Check if the AMI requires specific instance types (GPU, memory-optimized)
- Review CloudTrail for RunInstances errors

## Phase 2 — Remediate

MUST:
- Ensure active Marketplace subscription before launching
- Use a compatible instance type for the AMI architecture
- Launch in a region where the AMI is available

SHOULD:
- Request service quota increases if needed: `aws service-quotas request-service-quota-increase --service-code ec2 --quota-code L-1216C47A --desired-value <value>`
- Configure proper IAM instance profile for the application

MAY:
- Use launch templates for consistent deployments
- Set up Auto Scaling groups for production workloads

## Safety Ratings

```
safety_ratings:
  - "marketplace-entitlement get-entitlements: GREEN — read-only check"
  - "ec2 describe-instances: GREEN — read-only enumeration"
  - "ec2 describe-images: GREEN — read-only AMI check"
  - "ec2 run-instances: RED — launches billable compute resources"
  - "service-quotas request-service-quota-increase: YELLOW — requests limit change"
```

## Escalation Conditions

- AMI consistently fails to launch across multiple instance types and regions
- AMI requires instance types not available in the required region
- Service quota increase requests are denied
- AMI software fails to initialize after successful EC2 launch

## Data Sensitivity

- `aws ec2 describe-instances`: MEDIUM — reveals instance metadata and network configuration
- `aws ec2 describe-images`: LOW — returns AMI metadata
- Instance user data: HIGH — may contain initialization scripts and credentials
- Application logs on instance: HIGH — may contain sensitive business data

## Prohibited Actions

- NEVER launch instances in production VPCs without approval
- NEVER use overly permissive security groups (0.0.0.0/0 on all ports)
- NEVER embed credentials in instance user data without encryption
- NEVER launch large or expensive instance types without cost acknowledgment

## Phase 3 — Rollback

1. Terminate failed instances: `aws ec2 terminate-instances --instance-ids <instance-id>`
2. Release any allocated Elastic IPs: `aws ec2 release-address --allocation-id <alloc-id>`
3. Delete unused security groups created for the launch
4. Cancel pending service quota increase requests if no longer needed
5. Verify no EBS volumes remain from terminated instances: `aws ec2 describe-volumes --filters "Name=status,Values=available"`

## Common Issues

- symptoms: "RunInstances returns OptInRequired error"
  diagnosis: "No active Marketplace subscription for this AMI product."
  resolution: "Subscribe to the product in Marketplace console before launching the AMI."

## Output Format

```yaml
root_cause: "ami_launch_error — <specific_cause>"
evidence:
  - type: launch_failure
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix launch configuration"
  long_term: "Use launch templates and IaC for consistent deployments"
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

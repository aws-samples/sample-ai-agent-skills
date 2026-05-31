---
title: "B1 — Launch Failures"
description: "Diagnose why Auto Scaling is failing to launch new instances"
status: active
severity: HIGH
triggers:
  - "launch failed"
  - "instance not launching"
  - "scaling activity failed"
  - "InsufficientInstanceCapacity"
  - "launch error"
owner: devops-agent
objective: "Identify why instance launches are failing and resolve the issue"
context: "Launch failures occur due to: IAM permission issues, instance type unavailable in AZ, subnet capacity exhausted, security group not found, key pair missing, EBS volume limits, or service quota exceeded."
---

## Phase 1 — Triage

MUST:
- Check failed scaling activities: `aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg-name> --query 'Activities[?StatusCode==`Failed`].{Time:StartTime,Status:StatusCode,Message:StatusMessage}'`
- Check launch template/config: `aws ec2 describe-launch-template-versions --launch-template-id <lt-id> --versions '$Latest'`
- Verify instance type availability: `aws ec2 describe-instance-type-offerings --location-type availability-zone --filters Name=instance-type,Values=<type> Name=location,Values=<az>`
- Check service quotas: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A`

SHOULD:
- Verify subnet has available IPs: `aws ec2 describe-subnets --subnet-ids <subnet-id> --query 'Subnets[*].AvailableIpAddressCount'`
- Check security group exists: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Verify AMI exists and is available: `aws ec2 describe-images --image-ids <ami-id>`
- Check IAM instance profile: `aws iam get-instance-profile --instance-profile-name <profile-name>`

MAY:
- Check EBS volume limits
- Verify key pair exists if specified
- Check placement group constraints

## Phase 2 — Remediate

MUST:
- Fix the specific launch failure cause identified in StatusMessage
- If capacity unavailable: add more instance types or AZs
- If IAM issue: fix the service-linked role or instance profile permissions
- If resource not found: update launch template with valid resources

SHOULD:
- Use mixed instances policy for better capacity availability
- Distribute across multiple AZs
- Request quota increases proactively

MAY:
- Set up CloudWatch alarms on failed scaling activities
- Implement launch template versioning for rollback

## Output Format

```yaml
root_cause: "launch_failure — <specific_cause>"
evidence:
  - type: scaling_activity
    content: "<failed activity StatusMessage>"
  - type: resource_check
    content: "<resource availability>"
severity: HIGH
mitigation:
  immediate: "Fix the specific launch blocker"
  long_term: "Use mixed instances and multi-AZ for resilience"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix specific launch failure cause | YELLOW | Depends on the specific fix — review each change |
| Add more instance types or AZs | YELLOW | Changes instance diversity — affects cost and availability |
| Fix service-linked role or instance profile | YELLOW | IAM change — review scope |
| Update launch template with valid resources | YELLOW | Changes instance configuration |
| Use mixed instances policy | YELLOW | Changes allocation strategy |
| Request quota increases | GREEN | No immediate infrastructure change |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Launch template changes affect all new instances in the ASG
- IAM role changes may affect running instances

## Data Sensitivity

- Launch template reveals instance configuration including user data scripts
- IAM instance profiles reveal application permissions
- Subnet and security group configurations reveal network architecture

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER modify launch templates without testing in a non-production environment first
- NEVER change security groups without understanding network access implications

## Phase 3 — Rollback

- If launch template was updated: create a new version reverting changes, update ASG to use previous version
- If instance types were changed: restore original instance type configuration
- If IAM roles were modified: revert IAM policy changes
- If quota increase was unnecessary: no rollback needed (quotas don't auto-decrease)
- Document all launch configuration changes for operational audit trail

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

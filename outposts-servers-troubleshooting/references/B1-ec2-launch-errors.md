---
title: "B1 — EC2 Launch Errors"
description: "Diagnose EC2 instance launch failures on Outposts Servers"
status: active
severity: HIGH
triggers: ["instance launch failed on outpost", "InsufficientInstanceCapacity", "cannot launch EC2 on outpost server", "unsupported instance type"]
owner: devops-agent
objective: "Resolve EC2 instance launch failures specific to Outposts Servers"
context: "EC2 launches on Outposts Servers require an Outpost-associated subnet, supported instance type, available capacity, and active service link."
---

## Phase 1 — Triage

MUST:
- Check supported instance types: `aws outposts get-outpost-instance-types --outpost-id <outpost-id>`
- Verify subnet is associated with Outpost: `aws ec2 describe-subnets --filters "Name=outpost-arn,Values=<outpost-arn>" --query 'Subnets[*].{SubnetId:SubnetId,AvailabilityZone:AvailabilityZone,OutpostArn:OutpostArn}'`
- Check Outpost connectivity: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Review recent launch attempts in CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances --max-results 10`
- Check existing instance count on the Outpost
- Verify AMI compatibility with the instance type architecture (ARM vs x86)

MAY:
- Check security group and network ACL configurations
- Review placement group constraints

## Phase 2 — Remediate

MUST:
- Use a supported instance type for the Outpost Server form factor
- Launch in a subnet associated with the Outpost ARN
- Ensure service link is active before launching

SHOULD:
- Use Graviton-compatible AMIs for c6gd/m6gd/r6gd instance types
- Monitor capacity utilization before launching additional instances

MAY:
- Implement capacity reservation strategies
- Use launch templates with Outpost-specific configurations

## Common Issues

- symptoms: "Unsupported instance type error when launching on Outpost"
  diagnosis: "Instance type not available on Outposts Server form factor."
  resolution: "Check supported types: aws outposts get-outpost-instance-types --outpost-id <id>. Use c6gd/m6gd/r6gd (Graviton) or c6id/m6id/r6id (Intel)."

## Safety Ratings

```
safety_ratings:
  - "get-outpost-instance-types: GREEN — read-only capacity check"
  - "describe-subnets: GREEN — read-only subnet enumeration"
  - "RunInstances on Outpost: YELLOW — creates instances consuming limited server capacity"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- InsufficientInstanceCapacity errors despite expected available capacity
- Launch failures with internal service errors after verifying all prerequisites
- AMI compatibility issues that cannot be resolved with available AMIs
- Capacity exhaustion requiring additional Outposts Server hardware

## Data Sensitivity

- `aws outposts get-outpost-instance-types`: LOW — returns supported instance types
- `aws ec2 describe-subnets`: LOW — returns subnet metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals launch parameters and IAM principals
- Instance user data: HIGH — may contain secrets or configuration data

## Prohibited Actions

- NEVER launch instances on an Outpost with an inactive service link for production workloads
- NEVER terminate running instances to free capacity without explicit approval
- NEVER use instance types not listed by get-outpost-instance-types
- NEVER assume x86 AMIs work on Graviton instance types or vice versa

## Phase 3 — Rollback

1. If instances were launched in error, terminate them: `aws ec2 terminate-instances --instance-ids <ids>`
2. If a subnet was incorrectly associated, note that subnet-Outpost association is set at creation time — create a new subnet if needed
3. If launch templates were modified, revert to the previous version
4. Verify remaining instances on the Outpost are unaffected
5. Document capacity state after rollback

## Output Format

```yaml
root_cause: "ec2_launch_error — <specific_cause>"
evidence:
  - type: launch_failure
    content: "<error details and instance type>"
severity: HIGH
mitigation:
  immediate: "Fix launch configuration"
  long_term: "Implement launch templates with validated Outpost settings"
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

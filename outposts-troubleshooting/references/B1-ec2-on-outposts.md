---
title: "B1 — EC2 on Outposts Issues"
description: "Diagnose EC2 instance problems specific to AWS Outposts"
status: active
severity: HIGH
triggers:
  - "EC2 on Outposts"
  - "instance launch failed outpost"
  - "outpost instance"
  - "compute outpost"
owner: devops-agent
objective: "Resolve EC2 instance launch, runtime, and configuration issues on Outposts"
context: "EC2 on Outposts supports a subset of instance types and features. Launch failures may be due to capacity, unsupported instance types, or service link issues. Runtime issues may relate to local networking or storage."
---

## Phase 1 — Triage

MUST:
- Check instance status: `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[*].Instances[*].{State:State.Name,Type:InstanceType,OutpostArn:OutpostArn}'`
- Verify instance type is supported: `aws outposts get-outpost-instance-types --outpost-id <outpost-id>`
- Check capacity: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name InstanceFamilyCapacityAvailability --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check Outpost connectivity: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 60 --statistics Average`

SHOULD:
- Verify the subnet is associated with the Outpost: `aws ec2 describe-subnets --subnet-ids <subnet-id> --query 'Subnets[*].{SubnetId:SubnetId,OutpostArn:OutpostArn}'`
- Check security group and NACL rules
- Verify the AMI is available in the Region

MAY:
- Check placement group constraints if using placement groups
- Review instance metadata service configuration

## Phase 2 — Remediate

MUST:
- If unsupported instance type: use a supported type from `get-outpost-instance-types`
- If capacity exhausted: free capacity or use alternative instance family
- If service link down: wait for connectivity restoration for new launches

SHOULD:
- Ensure the launch template specifies an Outpost subnet
- Verify IAM permissions include ec2:RunInstances with Outpost resource conditions

MAY:
- Implement launch templates specific to Outpost deployments
- Set up capacity monitoring alerts

## Common Issues

- symptoms: "RunInstances fails with Unsupported error"
  diagnosis: "The requested instance type is not available on this Outpost."
  resolution: "Use aws outposts get-outpost-instance-types to find supported types."

- symptoms: "Instance launches but cannot reach the internet"
  diagnosis: "Outpost subnet route table lacks a NAT gateway or internet gateway route."
  resolution: "Add appropriate routes in the subnet route table pointing to Region NAT/IGW."

## Output Format

```yaml
root_cause: "ec2_outpost — <specific_cause>"
evidence:
  - type: instance_details
    content: "<instance configuration>"
  - type: capacity
    content: "<capacity availability>"
severity: HIGH
mitigation:
  immediate: "Resolve launch or runtime issue"
  long_term: "Implement Outpost-specific launch templates and monitoring"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — EC2 instance management is operational
- Service Disruption Risk: MEDIUM — instance issues affect individual workloads
- Compliance Impact: MEDIUM — EC2 instances on Outposts must meet data residency requirements

## Escalation Conditions
- EC2 instances fail to launch on Outpost
- Instance performance degradation without clear cause
- Instance connectivity issues specific to Outpost networking
- Instance placement constraints cannot be satisfied
- Instance metadata service issues on Outpost

## Data Sensitivity
- Instance configurations — INTERNAL
- Instance metadata — INTERNAL
- Workload data on instances — CONFIDENTIAL
- Security group configurations — INTERNAL
- Never expose instance credentials or user data

## Prohibited Actions
- Do NOT launch instances without verifying Outpost capacity
- Do NOT bypass placement group requirements
- Do NOT disable instance monitoring on production workloads
- Do NOT use unsupported instance types on Outposts
- Do NOT ignore instance health check failures
- Do NOT expose instance metadata endpoints to untrusted networks

## Phase 3 — Rollback
- Terminate failed instances and re-launch with correct configurations
- Restore previous security group rules if changes caused issues
- Revert instance type changes if they caused performance problems
- Restore instance placement configurations
- Verify instance connectivity after changes
- Document instance issues and resolutions

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

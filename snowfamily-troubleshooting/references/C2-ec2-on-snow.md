---
title: "C2 — EC2 on Snow"
description: "Diagnose EC2 instance issues on Snowball Edge devices"
status: active
severity: HIGH
triggers: ["EC2 on Snow error", "sbe1 instance failed", "AMI not found", "instance launch failed"]
owner: devops-agent
objective: "Resolve EC2 instance issues on Snowball Edge including AMI compatibility, instance types, and networking"
context: "EC2 on Snow uses sbe1 instance types with Snow-specific AMIs. Instances are managed via Snowball Edge client, not standard EC2 API."
---

## Phase 1 — Triage

MUST:
- List EC2 instances on device: `snowballEdge list-ec2-instances --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>`
- Check available AMIs: `snowballEdge describe-ec2-images --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>`
- Verify EC2 service is running: `snowballEdge describe-service --service-id ec2 --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>`

SHOULD:
- Check instance status and state
- Verify virtual network interface configuration

MAY:
- Review instance console output for boot errors
- Check device compute capacity for the requested instance type

## Phase 2 — Remediate

MUST:
- Use sbe1 instance types only (sbe1.small through sbe1.4xlarge)
- Ensure AMI was included in the job creation (AMIs cannot be added after device ships)
- Configure virtual network interface for instance networking

SHOULD:
- Verify AMI is compatible with the device type
- Check available compute capacity before launching instances

MAY:
- Use security groups for instance network access control
- Configure instance user data for automated setup

## Safety Ratings

```
safety_ratings:
  - "list-ec2-instances: GREEN — read-only instance listing"
  - "describe-ec2-images: GREEN — read-only AMI listing"
  - "run-ec2-instances: YELLOW — launches compute instances on device"
  - "terminate-ec2-instances: RED — destroys running instances and data"
```

## Escalation Conditions

- Required AMI was not included in the job and device is already on-site
- Instance type requires more resources than available on the device
- Instance networking issues prevent communication with other services
- Instance data persistence issues after device reboot

## Data Sensitivity

- `snowballEdge list-ec2-instances`: MEDIUM — reveals running workloads
- `snowballEdge describe-ec2-images`: LOW — reveals available AMIs
- Instance console output: HIGH — may contain application logs and credentials

## Prohibited Actions

- NEVER use standard EC2 instance types (t3, m5, etc.) — only sbe1 types work
- NEVER terminate instances without confirming data is backed up
- NEVER launch instances that exceed device compute capacity
- NEVER assume standard EC2 features (EBS, ELB, Auto Scaling) are available on Snow

## Phase 3 — Rollback

1. If an instance was launched with wrong configuration, terminate it: `snowballEdge terminate-ec2-instances --instance-id <id>`
2. If virtual network interface was misconfigured, delete and recreate it
3. Verify other instances on the device are not affected
4. Check device compute capacity after terminating instances
5. Document instance configuration for correct relaunch

## Common Issues

- symptoms: "RunInstances returns 'InvalidAMIID.NotFound'"
  diagnosis: "AMI was not included in the Snow job creation or AMI ID is incorrect."
  resolution: "AMIs must be specified during job creation. If missing, create a new job with the required AMIs."

## Output Format

```yaml
root_cause: "ec2_on_snow — <specific_cause>"
evidence:
  - type: ec2_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix EC2 instance issue"
  long_term: "Pre-validate AMIs in job creation"
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

---
title: "A2 — EFA Attachment Errors"
description: "Diagnose EFA network interface attachment failures"
status: active
severity: HIGH
triggers: ["efa attach failed", "cannot attach efa", "attachment error", "efa not attaching"]
owner: devops-agent
objective: "Resolve EFA interface attachment failures to EC2 instances"
context: "EFA interfaces must be attached at launch or while the instance is stopped. Hot-attach to running instances is not supported."
---

## Phase 1 — Triage

MUST:
- Check instance state: `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[0].Instances[0].State.Name'`
- Verify EFA interface exists and is available: `aws ec2 describe-network-interfaces --network-interface-ids <eni-id> --query 'NetworkInterfaces[0].{Status:Status,Type:InterfaceType,AZ:AvailabilityZone}'`
- Check current interface count vs maximum: `aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[0].NetworkInfo.{MaxInterfaces:MaximumNetworkInterfaces,MaxEfa:EfaInfo.MaximumEfaInterfaces}'`

SHOULD:
- Verify the EFA interface and instance are in the same AZ
- Check if the device index is already in use

MAY:
- Review CloudTrail for AttachNetworkInterface errors

## Phase 2 — Remediate

MUST:
- Stop the instance before attaching EFA: `aws ec2 stop-instances --instance-ids <instance-id>`
- Attach EFA interface: `aws ec2 attach-network-interface --network-interface-id <eni-id> --instance-id <instance-id> --device-index <index>`
- Start the instance: `aws ec2 start-instances --instance-ids <instance-id>`

SHOULD:
- Use launch templates with EFA pre-configured to avoid attachment issues
- Verify the EFA interface and instance are in the same subnet/AZ

MAY:
- Configure delete-on-termination for the EFA interface

## Safety Ratings

```
safety_ratings:
  - "ec2 describe-instances: GREEN — read-only instance status check"
  - "ec2 stop-instances: RED — stops the instance, disrupts workloads"
  - "ec2 attach-network-interface: YELLOW — modifies instance networking"
  - "ec2 start-instances: YELLOW — starts instance, may change public IP"
```

## Escalation Conditions

- Instance cannot be stopped due to running critical workloads
- EFA attachment fails even with instance in stopped state
- AZ mismatch between EFA interface and instance cannot be resolved
- Multiple EFA interfaces needed but device index conflicts exist

## Data Sensitivity

- `aws ec2 describe-instances`: MEDIUM — reveals instance configuration
- `aws ec2 describe-network-interfaces`: MEDIUM — reveals VPC topology
- `aws cloudtrail lookup-events`: MEDIUM — reveals API actions and principals
- Device index information: LOW — numeric interface ordering

## Prohibited Actions

- NEVER attempt to hot-attach EFA to a running instance — it will fail
- NEVER stop production instances without coordinating with the workload team
- NEVER attach EFA interfaces across different availability zones
- NEVER remove the primary network interface (device index 0) to make room for EFA

## Phase 3 — Rollback

1. Detach the EFA interface if attached incorrectly: `aws ec2 detach-network-interface --attachment-id <attachment-id> --force`
2. Start the instance if it was stopped: `aws ec2 start-instances --instance-ids <instance-id>`
3. Delete orphaned EFA interfaces: `aws ec2 delete-network-interface --network-interface-id <eni-id>`
4. Revert launch template if it was modified
5. Verify the instance returns to its previous working state

## Common Issues

- symptoms: "AttachNetworkInterface returns InvalidParameterValue for EFA interface"
  diagnosis: "Instance is in running state. EFA interfaces cannot be hot-attached."
  resolution: "Stop the instance, attach the EFA interface, then start the instance."

- symptoms: "EFA interface attachment succeeds but device is not visible in the OS"
  diagnosis: "EFA driver is not installed or the interface requires a reboot to be recognized."
  resolution: "Install the EFA driver package and reboot the instance."

## Output Format

```yaml
root_cause: "efa_attachment — <specific_cause>"
evidence:
  - type: attachment_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix attachment procedure"
  long_term: "Pre-configure EFA in launch templates"
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

---
title: "G1 — ENI (Elastic Network Interface) Issues"
description: "Diagnose ENI attachment, detachment, and configuration issues"
status: active
severity: MEDIUM
triggers:
  - "AttachNetworkInterface.*failed"
  - "ENI.*limit"
  - "network interface.*error"
owner: devops-agent
objective: "Resolve ENI issues and restore network connectivity"
context: "Each instance type has a maximum number of ENIs and IPs per ENI. ENIs are AZ-specific. Primary ENI cannot be detached. Secondary ENIs can be moved between instances in the same AZ/subnet."
---

## Phase 1 — Triage

MUST:
- Check ENI limits for instance type: `aws ec2 describe-instance-types --instance-types <type>` → NetworkInfo
- List attached ENIs: `aws ec2 describe-network-interfaces --filters Name=attachment.instance-id,Values=<id>`
- Check ENI state and attachment status

SHOULD:
- Verify ENI and instance are in the same subnet/AZ
- Check if source/destination check needs to be disabled (NAT, forwarding)

## Common Issues

- symptoms: "Cannot attach additional ENI"
  diagnosis: "Instance type ENI limit reached."
  resolution: "Detach unused ENIs or upgrade instance type."

- symptoms: "ENI stuck in 'attaching' or 'detaching'"
  diagnosis: "Underlying attachment issue. May require force detach."
  resolution: "Wait 5 minutes. If stuck, force detach: `aws ec2 detach-network-interface --attachment-id <id> --force`"

- symptoms: "Secondary ENI attached but no connectivity"
  diagnosis: "OS needs configuration for the secondary ENI (route table, IP config)."
  resolution: "Configure OS-level routing for the secondary ENI. On Amazon Linux, use ec2-net-utils."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-network-interfaces, describe-instance-types: GREEN — read-only"
  - "Attach secondary ENI: YELLOW — changes network configuration, detachable"
  - "Detach secondary ENI: YELLOW — removes network path, re-attachable"
  - "Force detach ENI: RED — may cause data loss on in-flight connections"
  - "Disable source/destination check: YELLOW — changes traffic filtering, recoverable"
  - "Configure OS-level routing for secondary ENI: YELLOW — changes routing tables, recoverable"
```

## Escalation Conditions
- Primary ENI has issues (cannot be detached or replaced without instance termination)
- ENI is managed by another AWS service (ECS, Lambda, EKS) and cannot be manually modified
- Force detach is required on an ENI with active network connections
- ENI limit reached and no ENIs can be safely detached
- Secondary ENI configuration requires OS-level changes on a production instance

## Data Sensitivity
- HIGH: describe-network-interfaces (reveals private/public IPs, security groups, MAC addresses)
- MEDIUM: describe-instance-types (reveals ENI and IP limits per instance type)
- LOW: ENI attachment/detachment operations (infrastructure changes)

## Prohibited Actions
- NEVER suggest detaching the primary ENI (eth0) — it cannot be detached
- NEVER suggest force-detaching an ENI without warning about connection disruption
- NEVER suggest disabling source/destination check without understanding the use case
- NEVER suggest attaching an ENI from a different AZ (will fail)

## Phase 3 — Rollback
- If secondary ENI was attached: detach with `detach-network-interface` and optionally delete
- If source/destination check was disabled: re-enable with `modify-network-interface-attribute`
- If OS routing was configured: revert routing table changes via SSM or reboot
- If ENI was force-detached: re-attach to original instance if still needed

## Output Format

```yaml
root_cause: "<eni_limit|az_mismatch|os_config|stuck_state> — <detail>"
evidence:
  - type: eni_state
    content: "<describe-network-interfaces output>"
severity: MEDIUM
mitigation:
  immediate: "Fix ENI attachment or OS configuration"
  long_term: "Automate ENI management, use launch templates"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"

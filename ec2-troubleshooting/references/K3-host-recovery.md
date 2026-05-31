---
title: "K3 — Dedicated Host Recovery"
description: "Handle host-level failures for instances on Dedicated Hosts"
status: active
severity: HIGH
triggers:
  - "host.*failure"
  - "dedicated host.*impaired"
  - "host recovery"
owner: devops-agent
objective: "Recover instances from a failed Dedicated Host"
context: "Dedicated Hosts provide physical server isolation. If the host fails, instances are stopped. Host recovery (if enabled) automatically migrates instances to a new host. Without host recovery, manual intervention is needed."
---

## Phase 1 — Triage

MUST:
- Check host state: `aws ec2 describe-hosts --host-ids <host-id>` → State
- Check if host recovery is enabled: `aws ec2 describe-hosts` → HostRecovery
- Check instance states on the affected host

## Phase 2 — Remediate

MUST:
- If host recovery enabled: instances auto-migrate (may take time)
- If host recovery disabled: allocate new host, stop instances, modify host affinity, start on new host
- Verify instances are running on the new host

## Common Issues

- symptoms: "Dedicated Host state 'under-assessment' or 'released-permanent-failure'"
  diagnosis: "Host hardware failure. Instances are stopped."
  resolution: "If host recovery enabled, wait for auto-migration. Otherwise, allocate new host and migrate manually."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-hosts, describe-instances: GREEN — read-only"
  - "Wait for auto-recovery: GREEN — no action required"
  - "Allocate new Dedicated Host: YELLOW — cost increase, recoverable by releasing"
  - "Modify instance host affinity: YELLOW — changes host placement, recoverable"
  - "Stop+start instances on new host: YELLOW — brief downtime, changes public IP"
  - "Release failed Dedicated Host: RED — permanently releases the host allocation"
```

## Escalation Conditions
- Host recovery is disabled and manual migration is required
- Multiple Dedicated Hosts fail simultaneously
- No replacement Dedicated Host capacity available in the AZ
- Instances on the failed host have licensing tied to specific host IDs
- Host failure affects instances with instance store data

## Data Sensitivity
- MEDIUM: describe-hosts (reveals host allocation, instance placement, tenancy details)
- MEDIUM: describe-instances (reveals instance states and host affinity)
- LOW: Host recovery status (reveals recovery configuration)

## Prohibited Actions
- NEVER suggest releasing a Dedicated Host that still has running instances
- NEVER suggest disabling host recovery without understanding the licensing implications
- NEVER suggest moving instances to shared tenancy to work around host failure
- NEVER suggest ignoring host failure notifications

## Phase 3 — Rollback
- If instances were migrated to new host: stop, modify host affinity back to original host (if recovered)
- If new Dedicated Host was allocated: release if no longer needed
- If host recovery was enabled: disable if auto-migration is not desired
- If instances were stopped during migration: start on the new host after verification

## Output Format

```yaml
root_cause: "Dedicated Host failure"
evidence:
  - type: host_state
    content: "<describe-hosts output>"
severity: HIGH
mitigation:
  immediate: "Wait for auto-recovery or manually migrate to new host"
  long_term: "Enable host recovery, use host resource groups for automation"
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

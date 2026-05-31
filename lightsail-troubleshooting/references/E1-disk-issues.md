---
title: "E1 — Disk Issues"
description: "Diagnose Lightsail additional disk attachment and capacity problems"
status: active
severity: MEDIUM
triggers:
  - "disk error"
  - "disk full"
  - "cannot attach disk"
  - "disk not available"
owner: devops-agent
objective: "Identify and resolve Lightsail disk issues"
context: "Lightsail supports additional block storage disks (8 GB to 32 TB) that can be attached to instances. Disks must be in the same AZ as the instance. Each instance can have up to 15 additional disks. Disks persist independently of instances."
---

## Phase 1 — Triage

MUST:
- List disks: `aws lightsail get-disks --query 'disks[*].{Name:name,Size:sizeInGb,State:state,Attached:isAttached,Instance:attachedTo,AZ:location.availabilityZone}'`
- Check disk state and attachment status
- Verify disk and instance are in the same AZ
- Check instance disk usage via SSH

SHOULD:
- Check if disk limit reached (15 per instance)
- Verify disk is formatted and mounted in the OS
- Check for I/O errors in system logs

MAY:
- Check disk performance metrics
- Review disk snapshot history

## Phase 2 — Remediate

MUST:
- Attach disk to instance: `aws lightsail attach-disk --disk-name <disk-name> --instance-name <instance-name> --disk-path /dev/xvdf`
- Format and mount new disks in the OS
- Ensure disk and instance are in the same AZ

SHOULD:
- Create disk snapshot before troubleshooting
- Expand disk if capacity is insufficient (requires snapshot + new disk)

MAY:
- Automate disk mounting in fstab
- Set up disk usage monitoring and alerts

## Common Issues

- symptoms: "Cannot attach disk to instance"
  diagnosis: "Disk and instance in different AZs or disk already attached."
  resolution: "Verify same AZ. Detach from other instance if needed."

- symptoms: "Disk attached but not visible in OS"
  diagnosis: "Disk not formatted or mounted."
  resolution: "Format disk (mkfs) and mount. Add to fstab for persistence."

- symptoms: "Disk full"
  diagnosis: "Data growth exceeded disk capacity."
  resolution: "Create snapshot, create larger disk from snapshot, attach new disk."

## Output Format

```yaml
root_cause: "disk_issue — <specific_cause>"
evidence:
  - type: disk_status
    content: "<disk state and attachment>"
  - type: capacity
    content: "<disk size and usage>"
severity: MEDIUM
mitigation:
  immediate: "Fix disk attachment or capacity issue"
  long_term: "Monitor disk usage and plan capacity"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete a static IP address attached to a production instance without reassignment planning
- **NEVER** modify firewall rules to allow unrestricted inbound access (0.0.0.0/0) on sensitive ports
- **NEVER** delete snapshots that serve as the only backup for production instances or databases
- **NEVER** force-stop a managed database instance without verifying active connections and transactions
- **NEVER** detach a disk from a running instance without first unmounting it from the operating system

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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

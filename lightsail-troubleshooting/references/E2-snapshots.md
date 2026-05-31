---
title: "E2 — Snapshot Issues"
description: "Diagnose snapshot creation, restore, and management problems"
status: active
severity: MEDIUM
triggers:
  - "snapshot failed"
  - "cannot create snapshot"
  - "restore from snapshot"
  - "snapshot error"
owner: devops-agent
objective: "Identify and resolve snapshot creation and restore issues"
context: "Lightsail supports instance snapshots (full instance) and disk snapshots (additional disks). Automatic snapshots retain 7 most recent. Manual snapshots have no auto-deletion. Snapshots can create new instances/disks in same or different regions. Snapshots are incremental after the first."
---

## Phase 1 — Triage

MUST:
- List instance snapshots: `aws lightsail get-instance-snapshots --query 'instanceSnapshots[*].{Name:name,State:state,Source:fromInstanceName,Created:createdAt}'`
- List disk snapshots: `aws lightsail get-disk-snapshots`
- Check automatic snapshot configuration: `aws lightsail get-auto-snapshots --resource-name <instance-name>`
- Verify snapshot state (available, error, etc.)

SHOULD:
- Check if snapshot creation is in progress
- Verify sufficient account snapshot limits
- Review snapshot sizes and costs

MAY:
- Check if instance was in a consistent state during snapshot
- Review cross-region snapshot copy status

## Phase 2 — Remediate

MUST:
- Create manual snapshot: `aws lightsail create-instance-snapshot --instance-name <name> --instance-snapshot-name <snap-name>`
- Restore from snapshot: `aws lightsail create-instances-from-snapshot --instance-names <new-name> --availability-zone <az> --bundle-id <bundle> --instance-snapshot-name <snap-name>`

SHOULD:
- Enable automatic snapshots: `aws lightsail enable-add-on --resource-name <instance-name> --add-on-request addOnType=AutoSnapshot,autoSnapshotAddOnRequest={snapshotTimeOfDay=06:00}`
- Test restore procedures regularly

MAY:
- Copy snapshots to other regions for DR
- Convert automatic snapshots to manual for long-term retention

## Common Issues

- symptoms: "Snapshot creation fails"
  diagnosis: "Instance in error state or account limit reached."
  resolution: "Ensure instance is running. Delete old snapshots if at limit."

- symptoms: "Instance from snapshot has wrong bundle"
  diagnosis: "Can only use same or larger bundle when creating from snapshot."
  resolution: "Specify a bundle equal to or larger than the original instance."

- symptoms: "Automatic snapshots not being created"
  diagnosis: "Auto snapshot add-on not enabled on the instance."
  resolution: "Enable auto snapshot add-on with desired time."

## Output Format

```yaml
root_cause: "snapshot — <specific_cause>"
evidence:
  - type: snapshot_status
    content: "<snapshot state>"
  - type: auto_snapshot
    content: "<automatic snapshot configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix snapshot creation or restore issue"
  long_term: "Enable automatic snapshots and test restore procedures"
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

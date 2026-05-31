---
title: "G1 — Snapshot Issues"
description: "Diagnose volume snapshot creation and management problems"
status: active
severity: HIGH
triggers:
  - "snapshot failed"
  - "snapshot not completing"
  - "cannot create snapshot"
  - "snapshot error"
owner: devops-agent
objective: "Identify and resolve volume snapshot issues"
context: "Volume gateway snapshots are stored as EBS snapshots in S3. Snapshots can be created on-demand or on a schedule. Snapshot creation requires the volume to be in a consistent state. Large volumes with many changes take longer to snapshot. Snapshots are incremental after the first full snapshot."
---

## Phase 1 — Triage

MUST:
- Check volume status: `aws storagegateway describe-cached-iscsi-volumes --volume-arns <volume-arn>` or stored equivalent
- List recent snapshots: `aws ec2 describe-snapshots --filters Name=volume-id,Values=<volume-id> --query 'Snapshots[*].{Id:SnapshotId,State:State,Progress:Progress}'`
- Check upload buffer utilization (snapshots require upload capacity)
- Verify gateway is online and healthy

SHOULD:
- Check CloudTrail for snapshot API errors
- Verify IAM permissions for snapshot creation
- Check if a snapshot is already in progress

MAY:
- Review snapshot schedule configuration
- Check EBS snapshot limits in the account

## Phase 2 — Remediate

MUST:
- Ensure gateway is online and volume is available
- Verify upload buffer has capacity for snapshot data
- Create snapshot: `aws storagegateway create-snapshot --volume-arn <arn> --snapshot-description "manual snapshot"`

SHOULD:
- Schedule regular snapshots: `aws storagegateway create-snapshot-from-volume-recovery-point --volume-arn <arn> --snapshot-description "recovery point"`
- Monitor snapshot progress and completion

MAY:
- Implement snapshot lifecycle policies to manage retention
- Use AWS Backup for centralized snapshot management

## Common Issues

- symptoms: "Snapshot creation fails"
  diagnosis: "Upload buffer full or gateway offline."
  resolution: "Check upload buffer. Ensure gateway is online. Retry after buffer drains."

- symptoms: "Snapshot takes very long"
  diagnosis: "Large volume with many changed blocks since last snapshot."
  resolution: "Expected for large volumes. First snapshot is full; subsequent are incremental."

- symptoms: "Cannot find snapshots in EC2 console"
  diagnosis: "Snapshots are in the same region as the gateway."
  resolution: "Check EC2 snapshots in the gateway's region. Filter by volume ID."

## Output Format

```yaml
root_cause: "snapshot_issue — <specific_cause>"
evidence:
  - type: volume_status
    content: "<volume state>"
  - type: snapshot_status
    content: "<snapshot state and progress>"
  - type: upload_buffer
    content: "<upload buffer utilization>"
severity: HIGH
mitigation:
  immediate: "Fix the specific snapshot blocker"
  long_term: "Implement automated snapshot schedules and monitoring"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Gateway volumes contain production application data
- Tape archives may contain regulated backup data
- File shares expose S3 bucket contents to on-premises systems
- Classification: CONFIDENTIAL - production data in transit and at rest

## Prohibited Actions

- NEVER shut down or reboot gateway VMs during active data transfers
- NEVER delete volumes or tapes containing unverified backup data
- NEVER modify cache disk configuration on production gateways without maintenance window
- NEVER change iSCSI targets or NFS/SMB shares during business hours without approval

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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

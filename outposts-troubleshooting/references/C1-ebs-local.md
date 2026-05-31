---
title: "C1 — EBS Local Volume Issues on Outposts"
description: "Diagnose EBS volume problems on AWS Outposts local storage"
status: active
severity: HIGH
triggers:
  - "EBS outpost"
  - "local volume"
  - "volume creation failed outpost"
  - "outpost storage"
owner: devops-agent
objective: "Resolve EBS volume creation, attachment, and performance issues on Outposts"
context: "EBS volumes on Outposts are stored on local hardware. Available volume types depend on the Outpost configuration. Snapshots are stored in the parent Region's S3. Volume capacity is limited by local storage."
---

## Phase 1 — Triage

MUST:
- List volumes on Outpost: `aws ec2 describe-volumes --filters "Name=outpost-arn,Values=<outpost-arn>" --query 'Volumes[*].{Id:VolumeId,Size:Size,Type:VolumeType,State:State}'`
- Check volume status: `aws ec2 describe-volume-status --volume-ids <volume-id>`
- Check available storage capacity on the Outpost
- Verify volume type is supported on the Outpost: `aws outposts get-outpost --outpost-id <outpost-id>`

SHOULD:
- Check volume attachment status: `aws ec2 describe-volumes --volume-ids <volume-id> --query 'Volumes[*].Attachments'`
- Review CloudWatch volume metrics: `aws cloudwatch get-metric-statistics --namespace AWS/EBS --metric-name VolumeReadOps --dimensions Name=VolumeId,Value=<volume-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check for snapshot creation issues

MAY:
- Review volume throughput and IOPS metrics
- Check for volume modification in progress

## Phase 2 — Remediate

MUST:
- If volume creation fails: verify local storage capacity and supported volume types
- If volume is impaired: create a snapshot and recreate the volume
- Ensure service link is up for new volume creation operations

SHOULD:
- Create regular snapshots for data protection (snapshots stored in Region S3)
- Monitor local storage utilization
- Use appropriate volume types for workload requirements

MAY:
- Implement automated snapshot schedules using AWS Backup
- Plan storage capacity expansion with AWS

## Common Issues

- symptoms: "CreateVolume fails with insufficient storage"
  diagnosis: "Local storage on the Outpost is exhausted."
  resolution: "Delete unused volumes or snapshots. Contact AWS for storage expansion."

- symptoms: "Volume performance is degraded"
  diagnosis: "Volume IOPS or throughput limits reached, or underlying hardware issue."
  resolution: "Check CloudWatch metrics. Consider volume type change or hardware investigation."

## Output Format

```yaml
root_cause: "ebs_local — <specific_cause>"
evidence:
  - type: volume_status
    content: "<volume state and health>"
  - type: storage_capacity
    content: "<available local storage>"
severity: HIGH
mitigation:
  immediate: "Resolve volume issue or restore from snapshot"
  long_term: "Implement snapshot schedules and storage monitoring"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — EBS volumes contain application data
- Service Disruption Risk: HIGH — storage failures cause data loss and application downtime
- Compliance Impact: HIGH — local storage must meet data residency and encryption requirements

## Escalation Conditions
- EBS volume creation fails on Outpost
- Volume performance degradation affecting applications
- Volume data corruption detected
- Snapshot operations fail for Outpost volumes
- Volume encryption key becomes inaccessible

## Data Sensitivity
- Volume data — CONFIDENTIAL
- Volume configurations — INTERNAL
- Snapshot data — CONFIDENTIAL
- Encryption key configurations — CONFIDENTIAL
- Never expose volume data or snapshot contents

## Prohibited Actions
- Do NOT create unencrypted EBS volumes on production Outposts
- Do NOT delete volumes without verifying backup availability
- Do NOT ignore volume performance degradation warnings
- Do NOT bypass volume encryption requirements
- Do NOT share snapshots with unauthorized accounts
- Do NOT detach volumes from running instances without proper shutdown

## Phase 3 — Rollback
- Restore volumes from snapshots if data corruption is detected
- Revert volume type changes if they caused performance issues
- Re-attach volumes if accidentally detached
- Restore encryption configurations
- Verify data integrity after volume recovery
- Document volume issues and update backup procedures

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

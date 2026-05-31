---
title: "D2 — Storage Issues"
description: "Diagnose Bottlerocket storage, volume mount, and disk space issues"
status: active
severity: HIGH
triggers: ["disk full", "volume mount failed", "storage error", "no space left", "ebs volume issues"]
owner: devops-agent
objective: "Resolve Bottlerocket storage issues including data partition, EBS volumes, and container storage"
context: "Bottlerocket has a read-only root partition and a separate data partition for container images and persistent data. The data partition is ext4 formatted."
---

## Phase 1 — Triage

MUST:
- Check instance storage status: `aws ec2 describe-instance-status --instance-ids <instance-id> --query 'InstanceStatuses[0].{InstanceStatus:InstanceStatus.Details,SystemStatus:SystemStatus.Details}'`
- Check EBS volume status: `aws ec2 describe-volumes --filters "Name=attachment.instance-id,Values=<instance-id>" --query 'Volumes[*].{VolumeId:VolumeId,Size:Size,State:State,Type:VolumeType}'`
- Check disk usage from admin container: `aws ssm start-session --target <instance-id> --document-name AWS-StartInteractiveCommand --parameters command="enter-admin-container"` then `sheltie` then `df -h`

SHOULD:
- Check CloudWatch disk metrics: `aws cloudwatch get-metric-statistics --namespace CWAgent --metric-name disk_used_percent --dimensions Name=InstanceId,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify container image storage usage: `sheltie` then `ctr -n k8s.io images list`

MAY:
- Check for orphaned container layers consuming disk space

## Phase 2 — Remediate

MUST:
- If data partition is full, identify and remove unused container images
- For EBS volume issues, verify volume is attached and in-use state
- Increase data volume size if needed: `aws ec2 modify-volume --volume-id <vol-id> --size <new-size>`

SHOULD:
- Configure container image garbage collection thresholds via settings API
- Set up CloudWatch alarms for disk usage monitoring

MAY:
- Add additional EBS volumes for persistent storage needs

## Safety Ratings

```
safety_ratings:
  - "ec2 describe-volumes: GREEN — read-only volume status check"
  - "df -h via sheltie: GREEN — read-only disk usage check"
  - "ec2 modify-volume: YELLOW — changes volume size, may incur costs"
  - "ctr image remove: RED — deletes container images, may affect running workloads"
```

## Escalation Conditions

- Data partition is 100% full and containers cannot start
- EBS volume shows impaired status in AWS health checks
- Volume modification fails or takes longer than expected
- Root partition corruption detected (dm-verity errors in console output)

## Data Sensitivity

- `aws ec2 describe-volumes`: LOW — returns volume metadata
- `df -h` output: LOW — returns disk usage statistics
- Container image list: LOW — returns image names and sizes
- `aws cloudwatch get-metric-statistics`: LOW — returns numeric metrics

## Prohibited Actions

- NEVER attempt to resize the root partition — it is immutable and dm-verity protected
- NEVER delete container images for running workloads without draining first
- NEVER format or partition EBS volumes attached to running instances
- NEVER suggest mounting volumes to paths managed by the Bottlerocket settings API

## Phase 3 — Rollback

1. If container images were removed in error, they will be re-pulled on next task/pod start
2. If EBS volume was modified, note that volume size cannot be decreased — only increased
3. If additional volume was attached incorrectly, detach: `aws ec2 detach-volume --volume-id <vol-id>`
4. If instance storage is corrupted, terminate and let auto scaling replace
5. Restore any persistent data from EBS snapshots if available

## Common Issues

- symptoms: "Containers fail to start with 'no space left on device'"
  diagnosis: "Data partition is full from accumulated container images and layers."
  resolution: "Clean unused images via admin container. Increase data volume size. Configure image GC thresholds."

- symptoms: "EBS volume attached but not visible inside Bottlerocket"
  diagnosis: "Additional EBS volumes require explicit mount configuration through settings API."
  resolution: "Use Kubernetes PersistentVolumes with the EBS CSI driver, or configure mounts via settings API."

## Output Format

```yaml
root_cause: "storage — <specific_cause>"
evidence:
  - type: storage_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Free disk space or fix volume attachment"
  long_term: "Configure image GC and disk monitoring"
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

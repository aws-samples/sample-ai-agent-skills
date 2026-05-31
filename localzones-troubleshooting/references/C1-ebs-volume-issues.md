---
title: "C1 — EBS Volume Issues"
description: "Diagnose EBS volume creation and attachment issues in Local Zones"
status: active
severity: HIGH
triggers: ["EBS volume error", "volume creation failed", "volume type not supported", "cannot attach volume"]
owner: devops-agent
objective: "Resolve EBS volume issues in Local Zones"
context: "Local Zones support a limited set of EBS volume types. Volumes must be in the same zone as the instance. Not all volume types (io2, sc1, st1) are available."
---

## Phase 1 — Triage

MUST:
- List volumes in the zone: `aws ec2 describe-volumes --filters "Name=availability-zone,Values=<local-zone-name>" --query 'Volumes[*].{VolumeId:VolumeId,VolumeType:VolumeType,State:State,Size:Size}'`
- Check volume type support: attempt to describe or create with desired type
- Verify volume and instance are in the same availability zone

SHOULD:
- Check volume status and attachment state
- Verify EBS encryption key availability in the region
- Check volume IOPS and throughput limits for the type

MAY:
- Review CloudTrail for CreateVolume or AttachVolume errors
- Check EBS volume quota for the account

## Phase 2 — Remediate

MUST:
- Use a supported volume type (gp2, gp3 are most commonly available)
- Create volume in the same Local Zone as the target instance
- Ensure KMS key is accessible if using encrypted volumes

SHOULD:
- Use gp3 for best price-performance in Local Zones
- Configure appropriate IOPS and throughput for gp3 volumes

MAY:
- Create snapshots for backup (snapshots are stored in parent region)
- Use instance store if available for temporary high-performance storage

## Safety Ratings

```
safety_ratings:
  - "describe-volumes: GREEN — read-only volume enumeration"
  - "create-volume: YELLOW — creates storage resource with cost"
  - "attach-volume: YELLOW — attaches storage to instance"
  - "delete-volume: RED — permanently deletes data"
  - "create-snapshot: YELLOW — creates backup with storage cost"
```

## Escalation Conditions

- Required volume type (io2) not available and workload needs high IOPS
- Volume creation fails with internal errors after fixing parameters
- Encrypted volume creation fails due to KMS key access issues
- Volume quota reached requiring limit increase

## Data Sensitivity

- Volume metadata: LOW — volume IDs, types, and sizes
- Volume contents: CRITICAL — may contain application data
- KMS key information: HIGH — encryption key ARNs and policies
- Snapshot data: HIGH — contains volume data copies

## Prohibited Actions

- NEVER delete EBS volumes without confirming data is backed up
- NEVER detach volumes from running instances without proper unmounting
- NEVER share snapshots of volumes containing sensitive data without encryption
- NEVER assume volume type availability without checking the specific Local Zone

## Phase 3 — Rollback

1. If volume was created with wrong type, create new volume with correct type and migrate data
2. If volume was detached incorrectly, reattach: `aws ec2 attach-volume --volume-id <id> --instance-id <id> --device <device>`
3. If volume was deleted, restore from snapshot: `aws ec2 create-volume --snapshot-id <snap-id> --availability-zone <local-zone>`
4. Verify data integrity after any volume operation

## Common Issues

- symptoms: "CreateVolume returns InvalidParameterValue for volume type io2"
  diagnosis: "io2 volume type is not supported in this Local Zone."
  resolution: "Use gp3 with configured IOPS as alternative. gp3 supports up to 16,000 IOPS."

- symptoms: "AttachVolume returns InvalidParameterValue for availability zone mismatch"
  diagnosis: "Volume and instance are in different availability zones."
  resolution: "Create volume in the same Local Zone as the instance. Volumes cannot be moved between zones."

## Output Format

```yaml
root_cause: "ebs_volume — <specific_cause>"
evidence:
  - type: volume_error
    content: "<volume details and error>"
severity: HIGH
mitigation:
  immediate: "Use supported volume type"
  long_term: "Document volume type availability per zone"
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

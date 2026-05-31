---
title: "B2 — EBS Local Volume Issues"
description: "Diagnose EBS local volume problems on Outposts Servers"
status: active
severity: HIGH
triggers: ["EBS volume failed on outpost", "cannot create volume on outpost", "volume stuck creating", "local storage error"]
owner: devops-agent
objective: "Resolve EBS volume creation, attachment, and performance issues on Outposts Servers"
context: "EBS on Outposts Servers uses local NVMe storage. Volumes do NOT replicate to the Region. Supported volume types are gp2 only. Snapshots require service link."
---

## Phase 1 — Triage

MUST:
- List EBS volumes on Outpost: `aws ec2 describe-volumes --filters "Name=volume-type,Values=gp2" --query 'Volumes[?OutpostArn!=null].{VolumeId:VolumeId,State:State,Size:Size,OutpostArn:OutpostArn}'`
- Check Outpost status: `aws outposts get-outpost --outpost-id <outpost-id> --query '{Status:LifeCycleStatus}'`
- Verify available storage capacity on the server: `aws outposts list-assets --outpost-identifier <outpost-id>`

SHOULD:
- Check for volume creation errors in CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateVolume --max-results 10`
- Verify the volume is in the same AZ as the Outpost subnet
- Check snapshot status if creating from snapshot

MAY:
- Review EBS CloudWatch metrics for performance issues
- Check instance-to-volume attachment limits

## Phase 2 — Remediate

MUST:
- Create volumes only in the Outpost-associated subnet's AZ with the Outpost ARN
- Use gp2 volume type (only supported type on Outposts Servers)
- Ensure service link is active for snapshot operations

SHOULD:
- Implement regular snapshot schedules to the Region for data protection
- Monitor local storage utilization to prevent capacity exhaustion

MAY:
- Use Amazon Data Lifecycle Manager for automated snapshot management
- Implement application-level replication for critical data

## Common Issues

- symptoms: "CreateVolume fails with InvalidParameterValue for volume type"
  diagnosis: "Unsupported volume type specified. Only gp2 is supported on Outposts Servers."
  resolution: "Use --volume-type gp2 and specify --outpost-arn when creating volumes."

## Safety Ratings

```
safety_ratings:
  - "describe-volumes: GREEN — read-only volume enumeration"
  - "CreateVolume on Outpost: YELLOW — consumes limited local storage"
  - "CreateSnapshot: YELLOW — creates snapshot, requires service link bandwidth"
  - "DeleteVolume: RED — permanently destroys local data with no recovery"
```

## Escalation Conditions

- Volume creation fails with internal errors despite correct parameters
- Local NVMe storage shows degradation or hardware errors
- Snapshot creation consistently fails even with active service link
- Storage capacity exhausted with no volumes available to delete

## Data Sensitivity

- `aws ec2 describe-volumes`: LOW — returns volume metadata
- `aws outposts list-assets`: LOW — returns hardware asset information
- EBS volume data: HIGH — may contain application data, databases, or secrets
- EBS snapshots: HIGH — contain full volume data copies

## Prohibited Actions

- NEVER delete EBS volumes on Outposts without confirming snapshots exist — local data is NOT recoverable
- NEVER assume EBS on Outposts has regional durability guarantees
- NEVER create io1/io2/gp3/st1/sc1 volumes on Outposts Servers — only gp2 is supported
- NEVER rely solely on local EBS for critical data without Region-based backups

## Phase 3 — Rollback

1. If a volume was deleted in error, restore from the most recent snapshot: `aws ec2 create-volume --snapshot-id <snap-id> --availability-zone <az> --outpost-arn <arn> --volume-type gp2`
2. If a volume is in error state, create a new volume and restore data from snapshot
3. If snapshot creation failed, attempt again with service link confirmed active
4. Document data loss if rollback is not possible (no snapshots available)
5. Review and update backup policies to prevent future data loss

## Output Format

```yaml
root_cause: "ebs_local_volume — <specific_cause>"
evidence:
  - type: volume_error
    content: "<volume state and error details>"
severity: HIGH
mitigation:
  immediate: "Resolve volume issue"
  long_term: "Implement automated snapshots and backup verification"
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

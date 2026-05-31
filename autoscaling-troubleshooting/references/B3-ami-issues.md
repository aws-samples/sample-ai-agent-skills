---
title: "B3 — AMI Issues"
description: "Diagnose AMI-related launch failures in Auto Scaling"
status: active
severity: HIGH
triggers:
  - "AMI not found"
  - "image not available"
  - "deregistered AMI"
  - "AMI permission denied"
  - "invalid AMI"
owner: devops-agent
objective: "Identify and resolve AMI issues preventing instance launches"
context: "AMI issues include: AMI deregistered/deleted, AMI in wrong region, cross-account AMI without launch permission, AMI architecture mismatch with instance type, or AMI based on deleted snapshot."
---

## Phase 1 — Triage

MUST:
- Get AMI from launch template: `aws ec2 describe-launch-template-versions --launch-template-id <lt-id> --versions '$Latest' --query 'LaunchTemplateVersions[*].LaunchTemplateData.ImageId'`
- Check AMI exists and state: `aws ec2 describe-images --image-ids <ami-id> --query 'Images[*].{Id:ImageId,State:State,Architecture:Architecture,Owner:OwnerId}'`
- Check failed scaling activities for AMI errors: `aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg-name> --query 'Activities[?StatusCode==`Failed`]'`

SHOULD:
- If cross-account AMI, check launch permissions: `aws ec2 describe-image-attribute --image-id <ami-id> --attribute launchPermission`
- Verify AMI architecture matches instance type (x86_64 vs arm64)
- Check if AMI snapshots still exist: `aws ec2 describe-images --image-ids <ami-id> --query 'Images[*].BlockDeviceMappings[*].Ebs.SnapshotId'`

MAY:
- Check if AMI was recently deregistered via CloudTrail
- Verify AMI is available in the correct region

## Phase 2 — Remediate

MUST:
- If AMI deleted: update launch template with valid AMI
- If permission issue: grant launch permission or copy AMI to your account
- If architecture mismatch: use compatible instance type or AMI

SHOULD:
- Copy cross-account AMI to own account: `aws ec2 copy-image --source-image-id <ami-id> --source-region <region> --name "copied-ami"`
- Update launch template: `aws ec2 create-launch-template-version --launch-template-id <lt-id> --source-version <ver> --launch-template-data '{"ImageId":"<new-ami>"}'`

MAY:
- Implement AMI lifecycle management with Image Builder
- Set up alerts for AMI deprecation

## Output Format

```yaml
root_cause: "ami_issue — <specific_cause>"
evidence:
  - type: ami_status
    content: "<AMI state and details>"
  - type: launch_template
    content: "<configured AMI ID>"
severity: HIGH
mitigation:
  immediate: "Update launch template with valid AMI"
  long_term: "Implement AMI lifecycle management and monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Update launch template with valid AMI | YELLOW | Changes instance image — test before production |
| Grant launch permission or copy AMI | YELLOW | Cross-account access change |
| Use compatible instance type or AMI | YELLOW | Changes instance configuration |
| Copy cross-account AMI to own account | GREEN | Creates local copy — non-destructive |
| Implement AMI lifecycle management | GREEN | Process improvement — non-destructive |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- AMI changes affect the operating system and software on all new instances
- Cross-account AMI access changes require security review

## Data Sensitivity

- AMI IDs reveal the software stack and operating system in use
- Cross-account AMI sharing reveals organizational relationships
- AMI contents may include pre-installed software with sensitive configurations

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER use untested AMIs in production launch templates
- NEVER deregister AMIs that are actively referenced by launch templates

## Phase 3 — Rollback

- If launch template AMI was changed: create new version with previous AMI, update ASG
- If AMI permissions were changed: revoke launch permissions if no longer needed
- If copied AMI is incorrect: deregister the copy and delete associated snapshots
- Document all AMI and template changes for operational audit trail

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

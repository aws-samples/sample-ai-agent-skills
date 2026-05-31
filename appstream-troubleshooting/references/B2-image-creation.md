---
title: "B2 — Image Creation Issues"
description: "Diagnose AppStream 2.0 image creation failures"
status: active
severity: MEDIUM
triggers:
  - "image creation failed"
  - "cannot create image"
  - "image stuck in pending"
  - "snapshot failed"
owner: devops-agent
objective: "Identify and resolve image creation failures"
context: "Images are created from image builders after installing applications. The image creation process snapshots the builder, validates applications, and creates the image. Failures can occur during snapshotting, application validation, or image finalization."
---

## Phase 1 — Triage

MUST:
- Check image status: `aws appstream describe-images --names <image-name>`
- Verify the image builder was in SNAPSHOTTING state during creation
- Check CloudTrail for image creation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateImage --max-results 10`
- Verify applications were properly configured on the image builder

SHOULD:
- Check if the image builder had sufficient disk space
- Verify all applications were tested before creating the image
- Check for application conflicts or missing dependencies

MAY:
- Review the Image Assistant logs on the image builder
- Check if the image name conflicts with an existing image

## Phase 2 — Remediate

MUST:
- Ensure applications are properly installed and tested on the image builder
- Use the Image Assistant to add applications before creating the image
- Verify sufficient disk space on the image builder

SHOULD:
- Test all applications on the image builder before creating the image
- Use descriptive image names with version numbers
- Document the applications and configuration in the image

MAY:
- Create images during off-peak hours
- Maintain a library of tested base images

## Common Issues

- symptoms: "Image creation fails during snapshotting"
  diagnosis: "Disk space issue or running processes interfering."
  resolution: "Close all applications. Ensure sufficient disk space. Retry image creation."

- symptoms: "Image created but applications not available"
  diagnosis: "Applications not added via Image Assistant before creating image."
  resolution: "Reconnect to image builder. Use Image Assistant to add applications. Create new image."

- symptoms: "Image stuck in PENDING state"
  diagnosis: "Image creation process taking longer than expected."
  resolution: "Wait up to 30 minutes. If still pending, check CloudTrail for errors."

## Output Format

```yaml
root_cause: "image_creation_issue — <specific_cause>"
evidence:
  - type: image_status
    content: "<image state>"
  - type: builder_status
    content: "<image builder state during creation>"
  - type: cloudtrail_events
    content: "<image creation events>"
severity: MEDIUM
mitigation:
  immediate: "Fix the image creation issue and retry"
  long_term: "Document image creation process and maintain image versioning"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure apps properly installed | GREEN | Verification |
| Use Image Assistant to add apps | GREEN | Image configuration |
| Verify sufficient disk space | GREEN | Capacity check |
| Test all applications | GREEN | Pre-creation validation |
| Create image | YELLOW | Creates new image from builder |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **Image contents**: Installed applications and configurations — INTERNAL
- **Image metadata**: Image name, version, applications list — INTERNAL
- **Builder state**: Snapshotting status — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER create images from builders with untested applications
- NEVER delete images that are actively used by fleets

## Phase 3 — Rollback

1. Delete the failed image if it was partially created
2. Reconnect to the image builder to fix application issues
3. Re-test all applications on the image builder
4. Create a new image with the corrected configuration
5. Keep the previous working image until the new one is validated

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

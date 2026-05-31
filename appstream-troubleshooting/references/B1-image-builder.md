---
title: "B1 — Image Builder Issues"
description: "Diagnose AppStream 2.0 image builder problems"
status: active
severity: MEDIUM
triggers:
  - "image builder not starting"
  - "cannot connect to image builder"
  - "image builder failed"
  - "image builder configuration"
owner: devops-agent
objective: "Identify and resolve image builder issues"
context: "Image builders are temporary instances for creating custom images. They must be in RUNNING state to connect. Connection is via the AppStream console (not SSH/RDP). Image builders need proper VPC configuration and IAM permissions."
---

## Phase 1 — Triage

MUST:
- Check image builder status: `aws appstream describe-image-builders --names <builder-name>`
- Verify the image builder state is RUNNING (not PENDING, STOPPED, or FAILED)
- Check IAM permissions for connecting to the image builder
- Verify VPC configuration if using custom networking

SHOULD:
- Check CloudTrail for image builder events
- Verify the base image is available
- Check if the instance type is available in the region

MAY:
- Check network connectivity from your browser to AppStream endpoints
- Verify browser compatibility for the AppStream console

## Phase 2 — Remediate

MUST:
- Start the image builder if stopped: `aws appstream start-image-builder --name <builder-name>`
- Fix VPC configuration (subnets, security groups)
- Ensure the base image is valid

SHOULD:
- Create image builder with proper configuration:
  `aws appstream create-image-builder --name <builder-name> --image-name <base-image> --instance-type <type> --vpc-config SubnetIds=<subnet>,SecurityGroupIds=<sg-id>`
- Connect via the AppStream console to install applications

MAY:
- Use a larger instance type for complex application installations
- Enable internet access for downloading application installers

## Common Issues

- symptoms: "Image builder stuck in PENDING state"
  diagnosis: "VPC networking issue or instance type unavailable."
  resolution: "Check VPC subnets, security groups, and NAT gateway configuration."

- symptoms: "Cannot connect to running image builder"
  diagnosis: "Browser or network issue blocking AppStream console connection."
  resolution: "Use a supported browser. Ensure HTTPS (443) is allowed to AppStream endpoints."

- symptoms: "Image builder fails to start"
  diagnosis: "Base image corrupted or instance type capacity issue."
  resolution: "Try a different base image or instance type."

## Output Format

```yaml
root_cause: "image_builder_issue — <specific_cause>"
evidence:
  - type: builder_status
    content: "<image builder state>"
  - type: vpc_config
    content: "<VPC configuration>"
  - type: base_image
    content: "<base image status>"
severity: MEDIUM
mitigation:
  immediate: "Fix image builder configuration"
  long_term: "Document image builder setup process"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Start image builder | GREEN | Non-production resource |
| Fix VPC configuration | YELLOW | Network configuration change |
| Ensure base image is valid | GREEN | Verification |
| Create image builder with config | GREEN | Non-production resource creation |
| Connect via AppStream console | GREEN | Console access |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **Image builder config**: Instance type, VPC, base image — INTERNAL
- **Application installers**: Software being installed — INTERNAL
- **Builder access**: Console connection details — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER install unlicensed software on image builders
- NEVER leave image builders running when not in use (cost impact)

## Phase 3 — Rollback

1. Stop the image builder: `aws appstream stop-image-builder --name <builder-name>`
2. Delete the image builder if no longer needed: `aws appstream delete-image-builder --name <builder-name>`
3. Revert VPC configuration changes if they were made
4. Use the previous base image for a new image builder
5. Document what was installed before the failure

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

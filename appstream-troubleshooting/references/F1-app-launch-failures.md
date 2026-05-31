---
title: "F1 — Application Launch Failures"
description: "Diagnose application launch failures within AppStream sessions"
status: active
severity: HIGH
triggers:
  - "application not launching"
  - "app crash in session"
  - "application error"
  - "app not available"
owner: devops-agent
objective: "Identify and resolve application launch failures in AppStream sessions"
context: "Application launch failures within sessions can be caused by missing dependencies, incorrect application paths in the image, insufficient instance resources, licensing issues, or application compatibility problems."
---

## Phase 1 — Triage

MUST:
- Verify the application is included in the image: `aws appstream describe-images --names <image-name> --query 'Images[*].Applications'`
- Check the fleet instance type has sufficient resources for the application
- Verify the application path is correct in the image configuration
- Check if the application requires specific dependencies or runtimes

SHOULD:
- Test the application on an image builder to reproduce the issue
- Check Windows Event Logs or application logs within the session
- Verify the application doesn't require a specific GPU (use Graphics instance type)

MAY:
- Check if the application requires network access that's blocked
- Verify licensing requirements are met

## Phase 2 — Remediate

MUST:
- Ensure the application is properly installed on the image
- Use the correct instance type for the application's resource requirements
- Fix application paths in the Image Assistant configuration

SHOULD:
- Test applications thoroughly on the image builder before creating images
- Include all required dependencies and runtimes in the image
- Use GPU instance types for graphics-intensive applications

MAY:
- Configure application-specific settings in the image
- Document application requirements and compatibility

## Common Issues

- symptoms: "Application not listed in session catalog"
  diagnosis: "Application not added via Image Assistant during image creation."
  resolution: "Create new image with the application added via Image Assistant."

- symptoms: "Application crashes immediately after launch"
  diagnosis: "Missing dependencies, insufficient memory, or compatibility issue."
  resolution: "Test on image builder. Install dependencies. Use larger instance type."

- symptoms: "Application requires GPU but session is slow"
  diagnosis: "Fleet using non-GPU instance type for graphics application."
  resolution: "Update fleet to use Graphics or GraphicsPro instance type."

## Output Format

```yaml
root_cause: "app_launch_failure — <specific_cause>"
evidence:
  - type: image_applications
    content: "<applications in the image>"
  - type: instance_type
    content: "<fleet instance type>"
  - type: error_details
    content: "<application error messages>"
severity: HIGH
mitigation:
  immediate: "Fix application configuration or instance type"
  long_term: "Standardize application testing before image creation"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure app properly installed on image | GREEN | Verification |
| Use correct instance type | YELLOW | Fleet configuration change |
| Fix application paths | YELLOW | Image configuration change |
| Test apps on image builder | GREEN | Non-production testing |
| Include dependencies in image | YELLOW | Image rebuild required |
| Use GPU instance types | YELLOW | Fleet configuration change |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **Image applications**: Installed software list — INTERNAL
- **Instance type**: Fleet compute configuration — INTERNAL
- **Application errors**: Error messages and crash details — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER modify application paths on a production image
- NEVER change instance types without testing application compatibility

## Phase 3 — Rollback

1. Revert to previous image on the fleet if image was changed
2. Revert instance type if it was modified
3. Verify all applications launch correctly with previous configuration
4. Communicate application availability to affected users
5. Keep the previous working image available for quick rollback

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

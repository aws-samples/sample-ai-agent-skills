---
title: "F1 — Greengrass Integration"
description: "Diagnose AWS IoT Greengrass V2 integration issues for edge model deployment"
status: active
severity: HIGH
triggers: ["Greengrass error", "edge device", "component deployment", "IoT integration", "Greengrass V2"]
owner: devops-agent
objective: "Resolve Greengrass integration issues for edge model deployment"
context: "Edge deployment of Lookout for Vision models requires Greengrass V2. The model is packaged as a Greengrass component and deployed to core devices. Greengrass manages the model lifecycle on edge."
---

## Phase 1 — Triage

MUST:
- List Greengrass core devices: `aws greengrassv2 list-core-devices --query 'coreDevices[*].{Name:coreDeviceThingName,Status:status}'`
- Check deployment status: `aws greengrassv2 list-deployments --query 'deployments[*].{Id:deploymentId,Name:deploymentName,Status:deploymentStatus}'`
- Verify the Lookout for Vision component exists: `aws greengrassv2 list-components --query 'components[*].{Name:componentName,Arn:arn}'`

SHOULD:
- Check core device connectivity: `aws greengrassv2 get-core-device --core-device-thing-name <name>`
- Verify deployment targets the correct core device or device group
- Check component recipe for configuration errors

MAY:
- Review Greengrass logs on the edge device
- Check IoT Core thing connectivity: `aws iot describe-thing --thing-name <name>`

## Phase 2 — Remediate

MUST:
- Ensure Greengrass V2 is installed and running on the edge device
- Verify the core device has HEALTHY status
- Fix component deployment configuration (correct component version, target)

SHOULD:
- Ensure edge device has sufficient resources (CPU, memory, disk) for the model
- Verify network connectivity between edge device and AWS IoT Core
- Check Greengrass nucleus version compatibility

MAY:
- Update Greengrass nucleus to the latest version
- Configure component environment variables for model inference settings
- Set up Greengrass log manager for remote log access

## Common Issues

- symptoms: "Deployment fails with component not found"
  diagnosis: "Model packaging job did not complete or component was created in different region."
  resolution: "Verify packaging job completed successfully. Check component exists in the same region as the Greengrass deployment."

- symptoms: "Component deploys but inference fails on edge device"
  diagnosis: "Edge device lacks resources or runtime dependencies."
  resolution: "Check device CPU, memory, and disk space. Verify required runtime (Python, OpenCV) is installed on the device."

## Safety Ratings

```
safety_ratings:
  - "greengrassv2 list-core-devices: GREEN — read-only enumeration"
  - "greengrassv2 list-deployments: GREEN — read-only enumeration"
  - "greengrassv2 create-deployment: YELLOW — deploys components to edge devices"
  - "greengrassv2 delete-deployment: YELLOW — removes deployment, may affect running components"
```

## Escalation Conditions

- Greengrass V2 installation fails on the edge device
- Edge device hardware is insufficient for the model
- Network connectivity between edge device and AWS is unreliable
- Multiple edge devices fail deployment simultaneously

## Data Sensitivity

- `aws greengrassv2 list-core-devices`: MEDIUM — reveals edge device identifiers
- `aws greengrassv2 list-deployments`: MEDIUM — reveals deployment configuration
- Greengrass logs: MEDIUM — may contain operational details
- Edge device configuration: MEDIUM — reveals device setup

## Prohibited Actions

- NEVER deploy untested components to production edge devices
- NEVER modify Greengrass nucleus configuration without understanding impact
- NEVER remove Greengrass from edge devices without coordinating with operations
- NEVER share edge device credentials or IoT certificates

## Phase 3 — Rollback

1. If deployment failed, roll back to previous deployment: create new deployment with previous component versions
2. If component is causing issues on edge device, remove it from the deployment
3. If Greengrass nucleus was updated and caused issues, roll back nucleus version
4. Verify edge inference is working after rollback
5. Document the integration issue for the edge operations team

## Output Format

```yaml
root_cause: "greengrass_integration — <specific_cause>"
evidence:
  - type: integration_error
    content: "<deployment status, device status, component details>"
severity: HIGH
mitigation:
  immediate: "Fix deployment configuration and retry"
  long_term: "Implement edge deployment pipeline with staging"
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

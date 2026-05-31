---
title: "C2 — Edge Deployment Failures"
description: "Diagnose model packaging and edge deployment failures"
status: active
severity: HIGH
triggers: ["edge deployment failed", "packaging error", "cannot deploy to edge", "Greengrass deployment"]
owner: devops-agent
objective: "Resolve edge deployment failures and deploy models to edge devices"
context: "Edge deployment requires packaging the model as a Greengrass V2 component, then deploying via Greengrass. The model must be trained and in TRAINED or HOSTED status."
---

## Phase 1 — Triage

MUST:
- Check model packaging job status: `aws lookoutvision list-model-packaging-jobs --project-name <project>`
- Verify model is trained: `aws lookoutvision describe-model --project-name <project> --model-version <version>`
- Check Greengrass core device status: `aws greengrassv2 list-core-devices --query 'coreDevices[*].{Name:coreDeviceThingName,Status:status}'`

SHOULD:
- Verify IAM role has permissions for lookoutvision, s3, and greengrassv2
- Check Greengrass component creation: `aws greengrassv2 list-components --query 'components[*].{Name:componentName,Arn:arn}'`

MAY:
- Review Greengrass deployment logs on the edge device
- Check CloudTrail for packaging job events

## Phase 2 — Remediate

MUST:
- Ensure model is in TRAINED status before packaging
- Fix IAM role permissions for lookoutvision:CreateModelPackagingJob and s3:PutObject
- Verify Greengrass V2 is installed and running on the target edge device

SHOULD:
- Ensure edge device meets hardware requirements for the model
- Verify network connectivity between edge device and AWS IoT Core

MAY:
- Test with a simpler model first to validate the deployment pipeline
- Check Greengrass component recipe for configuration errors

## Common Issues

- symptoms: "Model packaging job fails"
  diagnosis: "Model not in TRAINED status or IAM permissions insufficient."
  resolution: "Verify model status is TRAINED. Add lookoutvision:CreateModelPackagingJob and s3:PutObject to the service role."

- symptoms: "Greengrass deployment fails on edge device"
  diagnosis: "Edge device lacks resources or Greengrass V2 not properly configured."
  resolution: "Check device resources (CPU, memory, disk). Verify Greengrass V2 is running. Check deployment logs on the device."

## Safety Ratings

```
safety_ratings:
  - "lookoutvision list-model-packaging-jobs: GREEN — read-only enumeration"
  - "lookoutvision describe-model: GREEN — read-only model details"
  - "lookoutvision create-model-packaging-job: YELLOW — creates packaging job, incurs charges"
  - "greengrassv2 create-deployment: YELLOW — deploys components to edge devices"
```

## Escalation Conditions

- Packaging job fails with internal errors after fixing permissions
- Edge device hardware is insufficient for the model
- Greengrass V2 installation issues on the edge device
- Network connectivity between edge device and AWS is unreliable

## Data Sensitivity

- `aws lookoutvision list-model-packaging-jobs`: LOW — returns job metadata
- `aws greengrassv2 list-core-devices`: MEDIUM — reveals edge device identifiers
- Greengrass deployment logs: MEDIUM — may contain configuration details
- Model artifacts: HIGH — proprietary trained model

## Prohibited Actions

- NEVER deploy untested models to production edge devices
- NEVER modify Greengrass core device configuration without understanding impact
- NEVER share model packaging artifacts outside the organization
- NEVER deploy to edge devices without verifying hardware requirements

## Phase 3 — Rollback

1. If packaging job produced incorrect component, delete the Greengrass component version
2. If Greengrass deployment failed, roll back to previous deployment: `aws greengrassv2 create-deployment` with previous component versions
3. If edge device is in bad state, redeploy the last known good configuration
4. Verify edge inference is working after any rollback
5. Document the deployment failure for the edge operations team

## Output Format

```yaml
root_cause: "edge_deployment — <specific_cause>"
evidence:
  - type: deployment_error
    content: "<packaging job status, Greengrass deployment logs>"
severity: HIGH
mitigation:
  immediate: "Fix packaging/deployment and retry"
  long_term: "Implement edge deployment pipeline with validation"
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

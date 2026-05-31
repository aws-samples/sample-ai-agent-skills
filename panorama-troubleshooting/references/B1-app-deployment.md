---
title: "B1 — Application Deployment Failures"
description: "Diagnose application deployment issues on Panorama devices"
status: active
severity: HIGH
triggers:
  - "application deployment failed"
  - "DEPLOYMENT_ERROR"
  - "app not starting"
owner: devops-agent
objective: "Resolve application deployment failures on Panorama appliances"
context: "Application deployment downloads containers, verifies models, and starts the application graph. Takes 10-20 minutes."
---

## Phase 1 — Triage

MUST:
- Check app status: `aws panorama describe-application-instance --application-instance-id <id>`
- List app dependencies: `aws panorama list-application-instance-dependencies --application-instance-id <id>`
- Check device status: `aws panorama describe-device --device-id <device-id>`
- Review CloudWatch Logs: `/aws/panorama/devices/<device-id>`

SHOULD:
- Verify all referenced nodes exist: `aws panorama list-nodes`
- Check package versions: `aws panorama describe-package-version --package-id <id> --package-version <ver>`
- Verify device has sufficient resources for the application

MAY:
- Check for conflicting applications on the same device
- Verify S3 access for model artifacts

## Phase 2 — Remediate

MUST:
- Fix node references and package versions
- Ensure device is online and healthy
- Redeploy application after fixing issues

SHOULD:
- Remove failed application instances before redeploying
- Verify model compatibility with device hardware

## Common Issues

- symptoms: "DEPLOYMENT_ERROR with no specific message"
  diagnosis: "Model node incompatible or missing dependency."
  resolution: "Check all node references. Verify model was compiled for Panorama target."

- symptoms: "Deployment stuck in DEPLOYMENT_IN_PROGRESS"
  diagnosis: "Large model download or slow network."
  resolution: "Wait up to 30 minutes. Check network bandwidth. Review device logs."

## Output Format

```yaml
root_cause: "app_deployment — <specific_cause>"
evidence:
  - type: app_status
    content: "<application instance status>"
severity: HIGH
mitigation:
  immediate: "Fix deployment issue and redeploy"
  long_term: "Validate deployment manifests before deploying"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — deployed apps may process sensitive video data
- Service Disruption Risk: MEDIUM — deployment failures prevent new functionality
- Compliance Impact: MEDIUM — deployed apps must meet data processing requirements

## Escalation Conditions
- Application deployment fails with container errors
- Deployment causes device instability
- Application fails to start after deployment
- Deployment exceeds device resource limits
- Multiple application deployments fail simultaneously

## Data Sensitivity
- Application container images — INTERNAL
- Application configurations — INTERNAL
- Model artifacts — CONFIDENTIAL
- Video processing parameters — INTERNAL
- Never expose application source code or model weights

## Prohibited Actions
- Do NOT deploy untested applications to production devices
- Do NOT exceed device resource limits with deployments
- Do NOT deploy applications from untrusted container registries
- Do NOT bypass application validation before deployment
- Do NOT deploy applications that process data without proper consent
- Do NOT ignore deployment failure logs

## Phase 3 — Rollback
- Roll back to previous application version
- Remove failed application deployment
- Restore device to pre-deployment state
- Verify device health after rollback
- Re-deploy after fixing identified issues
- Document deployment failures and resolutions

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

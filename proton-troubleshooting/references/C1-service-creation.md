---
title: "C1 — Proton Service Creation Failures"
description: "Diagnose service and service instance creation failures"
status: active
severity: HIGH
triggers:
  - "service creation failed"
  - "service instance failed"
  - "cannot create service"
  - "service DEPLOYMENT_FAILED"
owner: devops-agent
objective: "Identify why a Proton service fails to create and resolve the deployment issue"
context: "Service creation deploys per-service infrastructure into an existing environment. Services reference environment outputs as inputs. Failures stem from template errors, missing environment outputs, IAM issues, or resource conflicts."
---

## Phase 1 — Triage

MUST:
- Check service status: `aws proton get-service --name <service-name> --query '{Status:status,StatusMessage:statusMessage}'`
- Check service instance status: `aws proton get-service-instance --name <instance-name> --service-name <service-name>`
- Verify environment is healthy: `aws proton get-environment --name <env-name> --query 'deploymentStatus'`
- Check environment outputs available: `aws proton list-environment-outputs --environment-name <env-name>`
- Check CloudFormation stack for service: `aws cloudformation describe-stack-events --stack-name <service-stack-name> --query 'StackEvents[?ResourceStatus==\`CREATE_FAILED\`]'`

SHOULD:
- Verify service template version is PUBLISHED
- Check service inputs match template schema requirements
- Verify service template is compatible with the environment template version

MAY:
- Check if service pipeline was created (if template includes pipeline)
- Review service template schema for required vs optional inputs

## Phase 2 — Remediate

MUST:
- Ensure environment is in IN_SERVICE status before creating service
- Provide all required inputs defined in the service template schema
- Fix template errors identified in deployment status message

SHOULD:
- Verify environment outputs match service template input expectations
- Test service creation in a staging environment first
- Check resource naming to avoid conflicts with existing services

MAY:
- Delete failed service and recreate after fixing issues
- Update environment if its outputs are missing required values

## Common Issues

- symptoms: "Service creation fails with missing input error"
  diagnosis: "Environment outputs don't include values expected by the service template."
  resolution: "Update environment template to produce required outputs, or fix service template inputs."

- symptoms: "Service instance DEPLOYMENT_FAILED"
  diagnosis: "CloudFormation stack creation failed for the service instance."
  resolution: "Check CloudFormation stack events for the specific resource failure."

- symptoms: "Service creation fails with template compatibility error"
  diagnosis: "Service template is not compatible with the environment template version."
  resolution: "Check compatibleEnvironmentTemplates in the service template version."

## Output Format

```yaml
root_cause: "service_creation_failure — <specific_cause>"
evidence:
  - type: service_status
    content: "<status and statusMessage>"
  - type: environment_outputs
    content: "<available environment outputs>"
  - type: stack_events
    content: "<CloudFormation failure events>"
severity: HIGH
mitigation:
  immediate: "Fix inputs, template compatibility, or resource issues"
  long_term: "Implement contract testing between environment and service templates"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — service creation deploys infrastructure
- Service Disruption Risk: MEDIUM — creation failures do not affect existing services
- Compliance Impact: MEDIUM — services must follow approved templates

## Escalation Conditions
- Service creation fails with template errors
- Service creation blocked by IAM policies
- Service creation leaves resources in inconsistent state
- Service creation timeout
- Multiple service creations fail simultaneously

## Data Sensitivity
- Service configurations — INTERNAL
- Deployment parameters — INTERNAL (may contain CONFIDENTIAL data)
- IAM role configurations — INTERNAL
- Resource details — INTERNAL
- Never expose service configurations externally

## Prohibited Actions
- Do NOT create services with unapproved templates
- Do NOT bypass service creation approval workflows
- Do NOT create services with overly permissive IAM roles
- Do NOT ignore service creation failures
- Do NOT create services without monitoring
- Do NOT include credentials in service parameters

## Phase 3 — Rollback
- Delete failed service resources
- Clean up orphaned infrastructure
- Restore IAM role configurations
- Document service creation failures
- Verify no residual resources remain
- Update service creation procedures

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

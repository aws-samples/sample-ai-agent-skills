---
title: "D1 — Proton Component Provisioning Issues"
description: "Diagnose component provisioning and management failures"
status: active
severity: MEDIUM
triggers:
  - "component provisioning failed"
  - "component error"
  - "component creation"
  - "customer-managed component"
owner: devops-agent
objective: "Identify why a Proton component fails to provision or update"
context: "Proton components are customer-managed infrastructure attached to service instances. Components extend service functionality without modifying templates. Component provisioning is the customer's responsibility — Proton tracks state but doesn't execute provisioning."
---

## Phase 1 — Triage

MUST:
- List components: `aws proton list-components`
- Check component status: `aws proton get-component --name <component-name>`
- Check component outputs: `aws proton list-component-outputs --component-name <component-name>`
- Verify associated service instance: check component's serviceInstanceName and serviceName
- Check CloudFormation stack if using CFN: `aws cloudformation describe-stack-events --stack-name <component-stack-name>`

SHOULD:
- Verify component template is valid
- Check component IAM role permissions
- Review component inputs match expected schema

MAY:
- Check if component conflicts with service instance resources
- Review component provisioning history

## Phase 2 — Remediate

MUST:
- Fix component template errors
- Grant required permissions to the component role
- Ensure component inputs are valid and complete

SHOULD:
- Test component templates independently before attaching to services
- Document component dependencies and requirements
- Use unique resource naming to avoid conflicts

MAY:
- Delete and recreate failed components
- Update component template to fix provisioning issues

## Common Issues

- symptoms: "Component creation fails with permission error"
  diagnosis: "Component role lacks permissions to create required resources."
  resolution: "Add missing permissions to the component IAM role."

- symptoms: "Component outputs not available to service instance"
  diagnosis: "Component provisioning incomplete or outputs not defined correctly."
  resolution: "Verify component is in a healthy state and outputs are defined in the template."

- symptoms: "Component conflicts with service instance resources"
  diagnosis: "Component creates resources that conflict with service-managed resources."
  resolution: "Review resource naming and ensure no overlaps between component and service."

## Output Format

```yaml
root_cause: "component_provisioning_issue — <specific_cause>"
evidence:
  - type: component_status
    content: "<component state and details>"
  - type: stack_events
    content: "<provisioning failure events>"
  - type: iam_role
    content: "<component role permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix template or permission issues and retry"
  long_term: "Standardize component templates and testing procedures"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — component provisioning creates infrastructure
- Service Disruption Risk: HIGH — provisioning failures affect service components
- Compliance Impact: MEDIUM — components must follow approved configurations

## Escalation Conditions
- Component provisioning fails with resource errors
- Provisioning leaves components in inconsistent state
- Component permissions prevent provisioning
- Component provisioning timeout
- Multiple components fail simultaneously

## Data Sensitivity
- Component configurations — INTERNAL
- Resource details — INTERNAL
- IAM role configurations — INTERNAL
- Provisioning logs — INTERNAL
- Never expose component configurations externally

## Prohibited Actions
- Do NOT force-delete failed components without cleanup
- Do NOT bypass component approval workflows
- Do NOT provision components with overly permissive roles
- Do NOT ignore component provisioning failures
- Do NOT modify components during active provisioning
- Do NOT provision components without monitoring

## Phase 3 — Rollback
- Delete failed component resources
- Restore previous component configurations
- Clean up orphaned resources
- Verify service health after component rollback
- Document component provisioning failures
- Update component templates based on findings

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

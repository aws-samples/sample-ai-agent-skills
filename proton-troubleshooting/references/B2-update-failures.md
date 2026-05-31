---
title: "B2 — Proton Environment Update Failures"
description: "Diagnose environment update and redeployment failures"
status: active
severity: HIGH
triggers:
  - "environment update failed"
  - "UPDATE_FAILED"
  - "cannot update environment"
  - "environment rollback"
owner: devops-agent
objective: "Identify why a Proton environment update fails and restore the environment to a healthy state"
context: "Environment updates include template version changes, input modifications, and redeployments. Updates use CloudFormation update stacks or Terraform apply. Failures can leave environments in an inconsistent state."
---

## Phase 1 — Triage

MUST:
- Check environment status: `aws proton get-environment --name <env-name> --query '{Status:deploymentStatus,Message:deploymentStatusMessage}'`
- Check current template version: `aws proton get-environment --name <env-name> --query '{Template:templateName,Major:templateMajorVersion,Minor:templateMinorVersion}'`
- For CloudFormation: `aws cloudformation describe-stack-events --stack-name <proton-stack-name> --query 'StackEvents[?ResourceStatus==\`UPDATE_FAILED\`]'`
- Check CloudTrail for update events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateEnvironment --max-results 10`

SHOULD:
- Compare old and new template version inputs/outputs
- Check if dependent services are affected by the update
- Verify no concurrent updates are in progress

MAY:
- Check CloudFormation change set for what was being modified
- Review drift detection results before the update

## Phase 2 — Remediate

MUST:
- If CloudFormation rollback succeeded, investigate the root cause before retrying
- Fix template or input issues identified in the failure message
- Verify dependent services are compatible with the new environment version

SHOULD:
- Test updates in a staging environment first
- Use minor version updates for non-breaking changes
- Plan major version updates with service team coordination

MAY:
- Cancel a stuck update if possible
- Recreate the environment if it's in an unrecoverable state

## Common Issues

- symptoms: "Environment update fails with resource modification error"
  diagnosis: "Template change requires replacing a resource that has dependencies."
  resolution: "Review CloudFormation events. May need to update dependent resources first."

- symptoms: "Environment stuck in IN_PROGRESS after update"
  diagnosis: "CloudFormation stack update is still running or stuck."
  resolution: "Check CloudFormation stack status. Wait or cancel if stuck."

- symptoms: "Update fails with input validation error"
  diagnosis: "New template version requires inputs not provided in the update."
  resolution: "Provide all required inputs for the new template version."

## Output Format

```yaml
root_cause: "environment_update_failure — <specific_cause>"
evidence:
  - type: deployment_status
    content: "<status and statusMessage>"
  - type: template_change
    content: "<old vs new template version>"
  - type: stack_events
    content: "<CloudFormation update failure events>"
severity: HIGH
mitigation:
  immediate: "Fix the identified issue and retry the update"
  long_term: "Test all template updates in staging before production"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — updates modify existing infrastructure
- Service Disruption Risk: HIGH — failed updates can leave environments in degraded state
- Compliance Impact: MEDIUM — updates must follow approved change procedures

## Escalation Conditions
- Update fails leaving environment in inconsistent state
- Update causes service disruption
- Update rollback fails
- Update blocked by resource dependencies
- Multiple updates fail simultaneously

## Data Sensitivity
- Update parameters — INTERNAL
- Resource state changes — INTERNAL
- CloudFormation stack details — INTERNAL
- Update logs — INTERNAL
- Never expose update parameters containing secrets

## Prohibited Actions
- Do NOT force-update environments in inconsistent state
- Do NOT bypass update approval workflows
- Do NOT update during peak usage without approval
- Do NOT ignore update failure patterns
- Do NOT modify resources outside Proton during updates
- Do NOT update without a rollback plan

## Phase 3 — Rollback
- Revert to previous environment version
- Restore resources to pre-update state
- Clean up partially updated resources
- Verify environment health after rollback
- Document update failures and root causes
- Update change management procedures

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

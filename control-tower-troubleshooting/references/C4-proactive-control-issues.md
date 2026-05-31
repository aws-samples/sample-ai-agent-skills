---
title: "C4 — Proactive Control Issues"
description: "Diagnose issues with proactive controls (CloudFormation hooks)"
status: active
severity: MEDIUM
triggers:
  - "CloudFormation hook failed"
  - "Proactive control blocking deployment"
  - "Hook validation error"
owner: devops-agent
objective: "Resolve proactive control issues blocking CloudFormation deployments"
context: "Proactive controls use CloudFormation hooks to validate resource configurations before deployment. They block non-compliant CF templates at deploy time. They only apply to CloudFormation deployments, not direct API calls."
---

## Phase 1 — Triage

MUST:
- Identify the proactive control that blocked the deployment
- Check CloudFormation stack events for hook failure details
- Review the resource configuration that violated the control
- Verify the control is enabled on the correct OU

SHOULD:
- Check if the control is in WARN or FAIL mode
- Review the CloudFormation template for the non-compliant resource
- Check if the control was recently enabled (may affect existing templates)

MAY:
- Test the template against the hook using CF template validation
- Check if the same resource can be created via direct API (proactive controls don't cover API calls)

## Phase 2 — Remediate

MUST:
- Fix the CloudFormation template to comply with the proactive control
- If the control is too restrictive: switch to WARN mode or disable it

SHOULD:
- Update all templates that may be affected by the control
- Document the control requirements for template authors
- Add pre-deployment validation to CI/CD pipelines

MAY:
- Create compliant template snippets or modules for common patterns
- Implement template linting that checks for proactive control compliance

## Common Issues

- symptoms: "CF deployment fails with hook validation error"
  diagnosis: "The template creates a resource that violates a proactive control."
  resolution: "Update the template to comply. Example: add encryption configuration, enable logging."

- symptoms: "Proactive control blocks but detective control doesn't flag"
  diagnosis: "Proactive controls validate CF templates; detective controls evaluate deployed resources. Different evaluation points."
  resolution: "This is expected. Fix the template to pass the proactive control."

- symptoms: "Direct API call succeeds but CF deployment fails"
  diagnosis: "Proactive controls only apply to CloudFormation deployments, not direct API calls."
  resolution: "This is by design. Use preventive controls (SCPs) to block direct API calls too."

## Safety Ratings

```
safety_ratings:
  - "describe-stack-events: GREEN — read-only stack event details"
  - "list-enabled-controls: GREEN — read-only control listing"
  - "switch proactive control to WARN mode: YELLOW — recoverable, can switch back to FAIL"
  - "disable proactive control: YELLOW — recoverable, control can be re-enabled"
  - "modify CloudFormation templates to comply: YELLOW — template changes are version-controlled"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Proactive control blocks critical production deployments with no workaround
- Control behavior changed after landing zone update

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- MEDIUM: OU structure, control enablement status
- MEDIUM: CloudFormation template content (expose infrastructure patterns)
- LOW: Proactive control configuration (WARN vs FAIL mode)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If a proactive control was switched to WARN mode to unblock deployment, plan to switch back to FAIL after templates are updated
- If a proactive control was disabled, document the justification and re-enable after template compliance is achieved
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- Keep the original CloudFormation template version before making compliance changes

## Output Format

```yaml
root_cause: "proactive_control — <control_id> blocking <resource_type>"
evidence:
  - type: cloudformation_events
    content: "<hook failure details>"
severity: MEDIUM
mitigation:
  immediate: "Fix the CF template to comply with the control"
  long_term: "Add pre-deployment validation and compliant template modules"
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
  - "NEVER suggest disabling mandatory controls"
  - "NEVER suggest removing the audit account"
  - "NEVER modify landing zone without understanding blast radius"

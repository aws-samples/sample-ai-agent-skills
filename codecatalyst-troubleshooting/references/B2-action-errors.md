---
title: "B2 — Action Errors"
description: "Diagnose workflow action execution errors"
status: active
severity: HIGH
triggers: ["action failed", "action error", "action timeout", "build action error"]
owner: devops-agent
objective: "Resolve individual workflow action failures"
context: "Actions are individual steps within workflows. They run in managed compute and have specific input/output contracts."
---

## Phase 1 — Triage

MUST:
- Get workflow run details: `aws codecatalyst get-workflow-run --space-name <space> --project-name <project> --id <run-id>`
- Identify the specific failing action from the run details
- Review action logs for error messages

SHOULD:
- Check action input parameters and environment variables
- Verify action version compatibility
- Review action dependencies (artifacts from previous actions)

MAY:
- Compare action configuration with documentation
- Check if the action has known issues or version-specific bugs

## Phase 2 — Remediate

MUST:
- Fix action input parameters if validation errors
- Update action version if using deprecated version
- Ensure required secrets and environment variables are configured

SHOULD:
- Add error handling and retry logic for flaky actions
- Configure action timeout appropriately
- Verify artifact passing between actions

MAY:
- Replace problematic actions with alternative implementations
- Add debugging output to action configuration

## Safety Ratings

```
safety_ratings:
  - "get-workflow-run: GREEN — read-only run details"
  - "action log review: GREEN — read-only log access"
  - "workflow YAML modification: YELLOW — changes action behavior"
  - "action re-execution: YELLOW — runs action with potentially modified inputs"
```

## Escalation Conditions

- Action fails with internal CodeCatalyst errors
- Action timeout cannot be resolved by increasing limits
- Action requires permissions not available in the workflow role
- Critical deployment action fails repeatedly

## Data Sensitivity

- `aws codecatalyst get-workflow-run`: LOW — returns run metadata
- Action logs: MEDIUM — may contain build output and environment details
- Action inputs: MEDIUM — may contain configuration parameters
- Action secrets: HIGH — encrypted credentials used by actions

## Prohibited Actions

- NEVER expose action secrets in logs or outputs
- NEVER bypass security scanning actions
- NEVER increase action timeouts beyond reasonable limits without investigation
- NEVER modify action inputs for production workflows without testing

## Phase 3 — Rollback

1. If action configuration was modified, restore previous workflow YAML
2. If action version was changed, revert to the previously working version
3. Re-run the workflow to verify the fix
4. Confirm downstream actions and deployments are not affected
5. Document the action error and resolution

## Common Issues

- symptoms: "Action fails with 'permission denied' error"
  diagnosis: "Workflow IAM role lacks permissions required by the action."
  resolution: "Update the IAM role policy to include permissions required by the action."

- symptoms: "Action times out after maximum duration"
  diagnosis: "Action is taking longer than configured timeout or is stuck."
  resolution: "Investigate why the action is slow. Optimize build steps or increase timeout if justified."

## Output Format

```yaml
root_cause: "action_error — <specific_cause>"
evidence:
  - type: action_log
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix action error"
  long_term: "Implement action testing and version pinning"
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

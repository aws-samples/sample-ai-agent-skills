---
title: "B1 — Workflow Failures"
description: "Diagnose workflow execution failures"
status: active
severity: HIGH
triggers: ["workflow failed", "workflow error", "pipeline broken", "CI/CD failure"]
owner: devops-agent
objective: "Resolve workflow execution failures and configuration issues"
context: "Workflows are YAML-defined CI/CD pipelines. They use CodeCatalyst-specific syntax, not GitHub Actions."
---

## Phase 1 — Triage

MUST:
- List recent workflow runs: `aws codecatalyst list-workflow-runs --space-name <space> --project-name <project> --sort-by START_TIME`
- Get failed run details: `aws codecatalyst get-workflow-run --space-name <space> --project-name <project> --id <run-id>`
- Check workflow definition for syntax errors

SHOULD:
- Review workflow run logs for specific error messages
- Verify workflow triggers are configured correctly
- Check if the failure is in a specific action or the overall workflow

MAY:
- Compare with a previously successful workflow run
- Check compute quota usage for the space

## Phase 2 — Remediate

MUST:
- Fix workflow YAML syntax errors
- Ensure referenced actions exist and are correctly specified
- Verify environment variables and secrets are configured

SHOULD:
- Update action versions to latest compatible versions
- Fix compute configuration (Fleet, instance type)
- Ensure source repository branch exists for trigger

MAY:
- Add workflow debugging steps (echo commands, artifact uploads)
- Configure workflow notifications for failure alerts

## Safety Ratings

```
safety_ratings:
  - "list-workflow-runs: GREEN — read-only run listing"
  - "get-workflow-run: GREEN — read-only run details"
  - "workflow YAML modification: YELLOW — changes CI/CD pipeline behavior"
  - "workflow re-run: YELLOW — executes pipeline actions"
```

## Escalation Conditions

- All workflows in the project failing simultaneously
- Workflow failures caused by CodeCatalyst service issues
- Workflow compute quota exhausted for the space
- Workflow failure blocks critical production deployment

## Data Sensitivity

- `aws codecatalyst list-workflow-runs`: LOW — returns run metadata
- `aws codecatalyst get-workflow-run`: LOW — returns run status and configuration
- Workflow logs: MEDIUM — may contain environment variables or build output
- Workflow YAML: MEDIUM — reveals CI/CD pipeline configuration

## Prohibited Actions

- NEVER hardcode secrets in workflow YAML files
- NEVER disable security scanning actions to fix workflow failures
- NEVER modify production deployment workflows without review
- NEVER share workflow logs containing secrets or credentials

## Phase 3 — Rollback

1. If workflow YAML was modified, restore previous version from source control
2. If workflow environment variables were changed, restore original values
3. Re-run the workflow to verify the fix
4. Confirm downstream deployments are not affected
5. Document the workflow failure and resolution

## Common Issues

- symptoms: "Workflow fails with 'action not found' error"
  diagnosis: "Action reference in workflow YAML is incorrect or action does not exist."
  resolution: "Verify action identifier matches a valid CodeCatalyst action. Check action catalog."

- symptoms: "Workflow fails at checkout step"
  diagnosis: "Source repository or branch reference is incorrect."
  resolution: "Verify repository name and branch exist. Check workflow trigger branch configuration."

## Output Format

```yaml
root_cause: "workflow_failure — <specific_cause>"
evidence:
  - type: workflow_run
    content: "<run details>"
severity: HIGH
mitigation:
  immediate: "Fix workflow failure"
  long_term: "Implement workflow testing and validation"
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

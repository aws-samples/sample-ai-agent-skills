---
title: "C4 — Lifecycle Hook Errors"
description: "Diagnose and resolve CodeDeploy lifecycle hook script failures"
status: active
severity: HIGH
triggers:
  - "lifecycle hook"
  - "hook failed"
  - "BeforeInstall"
  - "AfterInstall"
  - "ApplicationStart"
  - "ApplicationStop"
  - "script error"
owner: devops-agent
objective: "Identify why lifecycle hook scripts are failing and resolve the issue"
context: "CodeDeploy lifecycle hooks execute scripts at specific points during deployment. EC2 hooks: ApplicationStop → BeforeInstall → AfterInstall → ApplicationStart → ValidateService. Hook scripts must exit with code 0 for success. Non-zero exit codes fail the deployment at that lifecycle event. Scripts run as root by default."
---

## Phase 1 — Triage

MUST:
- Get deployment target lifecycle events: `aws codedeploy get-deployment-target --deployment-id <deployment-id> --target-id <instance-id>`
- Identify which lifecycle hook failed and its error message
- Check appspec.yml hook configuration for the failing hook
- Check hook script logs on the instance: `/opt/codedeploy-agent/deployment-root/<deployment-group-id>/<deployment-id>/logs/scripts.log`

SHOULD:
- Verify hook scripts exist in the deployment artifact at the paths specified in appspec.yml
- Check hook script permissions (must be executable)
- Verify hook scripts have correct shebang lines (#!/bin/bash, #!/usr/bin/env python, etc.)

MAY:
- Check CodeDeploy agent logs: `/var/log/aws/codedeploy-agent/codedeploy-agent.log`
- Review hook script timeout configuration in appspec.yml
- Test hook scripts manually on the instance

## Phase 2 — Remediate

MUST:
- If script not found: ensure the script path in appspec.yml matches the actual file path in the artifact
- If permission denied: make scripts executable (`chmod +x`) before packaging the artifact
- If script error: fix the script logic, check for missing dependencies or incorrect paths
- If timeout: increase the hook timeout in appspec.yml or optimize the script

SHOULD:
- Add error handling and logging to hook scripts for better debugging
- Use absolute paths in hook scripts (working directory may vary)
- Test hook scripts locally before deploying

MAY:
- Use environment variables provided by CodeDeploy: `DEPLOYMENT_ID`, `DEPLOYMENT_GROUP_NAME`, `APPLICATION_NAME`, `LIFECYCLE_EVENT`
- Implement idempotent hook scripts that can be safely re-run
- Add `set -e` to bash scripts to fail fast on any command error

## Common Issues

- symptoms: "BeforeInstall hook failed — script not found"
  diagnosis: "Script path in appspec.yml does not match the actual file location in the artifact."
  resolution: "Verify the script path in appspec.yml. Paths are relative to the artifact root."

- symptoms: "AfterInstall hook failed — permission denied"
  diagnosis: "Hook script is not executable."
  resolution: "Add `chmod +x` to the script before packaging, or add a `runas` directive in appspec.yml."

- symptoms: "ApplicationStart hook failed — service not starting"
  diagnosis: "The application service failed to start. Check service logs."
  resolution: "Review application logs. Verify configuration files were deployed correctly. Check port conflicts."

- symptoms: "Hook timed out after 3600 seconds"
  diagnosis: "Hook script exceeded the default timeout (1 hour)."
  resolution: "Optimize the script or increase timeout in appspec.yml: `timeout: 7200` under the hook definition."

- symptoms: "ApplicationStop hook failed on first deployment"
  diagnosis: "ApplicationStop runs before the new revision is installed. On first deployment, there's no previous application to stop."
  resolution: "Make ApplicationStop scripts handle the case where the application doesn't exist yet (check before stopping)."

## Output Format

```yaml
root_cause: "Lifecycle hook error — <hook_name> — <specific_cause>"
evidence:
  - type: lifecycle_events
    content: "<failed hook details>"
  - type: script_logs
    content: "<hook script output>"
severity: HIGH
mitigation:
  immediate: "Fix the hook script and create a new deployment"
  long_term: "Add error handling to hooks, test scripts locally, implement idempotent scripts"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| If script not found: ensure the script path in appspec.yml matches actual file path | GREEN |
| If permission denied: make scripts executable before packaging | GREEN |
| If script error: fix the script logic | YELLOW |
| If timeout: increase the hook timeout in appspec.yml or optimize the script | GREEN |
| Add error handling and logging to hook scripts | GREEN |
| Use absolute paths in hook scripts | GREEN |
| Test hook scripts locally before deploying | GREEN |
| Use environment variables provided by CodeDeploy | GREEN |
| Implement idempotent hook scripts | GREEN |
| Add set -e to bash scripts | GREEN |

## Escalation Conditions

- Fix requires modifying production pipeline
- Fix requires changing deployment strategy

## Data Sensitivity

- **HIGH**: build logs (may contain secrets), artifact contents
- Lifecycle hook scripts may contain deployment logic, service credentials, and infrastructure details

## Prohibited Actions

- NEVER suggest storing secrets in buildspec.yml
- NEVER suggest skipping approval stages in production pipelines

## Phase 3 — Rollback

1. If hook scripts were modified, revert changes in source control and create a new deployment
2. If appspec.yml was changed, restore the previous version in source control
3. If hook timeouts were increased, restore previous timeout values
4. Deploy the previous known-good revision: `aws codedeploy create-deployment --application-name <app> --deployment-group-name <dg> --revision <previous-revision>`
5. Verify rollback by confirming all lifecycle hooks pass on the next deployment

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

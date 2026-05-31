---
title: "C1 — Deployment Failures"
description: "Diagnose and resolve CodeDeploy deployment failures"
status: active
severity: CRITICAL
triggers:
  - "deployment failed"
  - "deployment error"
  - "DeploymentFailed"
  - "instance failed"
  - "appspec"
owner: devops-agent
objective: "Identify why the CodeDeploy deployment failed and resolve the issue"
context: "CodeDeploy deployments fail when lifecycle hooks error out, health checks fail, the CodeDeploy agent is not running, appspec.yml is misconfigured, or IAM permissions are insufficient. The deployment type (in-place vs blue/green) and compute platform (EC2, ECS, Lambda) determine the troubleshooting path."
---

## Phase 1 — Triage

MUST:
- Get deployment details: `aws codedeploy get-deployment --deployment-id <deployment-id>`
- Check deployment targets: `aws codedeploy list-deployment-targets --deployment-id <deployment-id>`
- Get instance-level status: `aws codedeploy get-deployment-target --deployment-id <deployment-id> --target-id <instance-id>`
- Check deployment group configuration: `aws codedeploy get-deployment-group --application-name <app> --deployment-group-name <dg>`

SHOULD:
- Check CodeDeploy agent status on target instances (via SSM): `aws ssm send-command --instance-ids <id> --document-name "AWS-RunShellScript" --parameters commands=["sudo service codedeploy-agent status"]`
- Verify appspec.yml is in the deployment artifact root
- Check the deployment configuration (min healthy hosts): `aws codedeploy get-deployment-config --deployment-config-name <config>`

MAY:
- Check CodeDeploy agent logs on the instance: `/var/log/aws/codedeploy-agent/codedeploy-agent.log`
- Review CloudTrail for deployment API errors
- Check if the deployment was stopped manually

## Phase 2 — Remediate

MUST:
- If agent not running: install/restart the CodeDeploy agent on target instances
- If appspec.yml missing: add appspec.yml to the deployment artifact root
- If lifecycle hook failed: fix the hook script (check exit codes, permissions, paths)
- If IAM permission error: update the CodeDeploy service role or instance profile

SHOULD:
- Verify all hook scripts are executable and have correct shebang lines
- Ensure the deployment artifact contains all required files referenced in appspec.yml
- Check that the instance profile has permissions for any AWS services the hooks call

MAY:
- Use `aws codedeploy create-deployment` with `--ignore-application-stop-failures` to skip ApplicationStop errors
- Roll back to the previous revision if the fix requires code changes

## Common Issues

- symptoms: "Deployment failed — InstanceAgent not running"
  diagnosis: "CodeDeploy agent is not installed or not running on the target instance."
  resolution: "Install agent: `sudo yum install -y codedeploy-agent` or use SSM to install. Start: `sudo service codedeploy-agent start`."

- symptoms: "Deployment failed — appspec.yml not found"
  diagnosis: "The appspec.yml file is missing from the deployment artifact root."
  resolution: "Add appspec.yml to the root of the deployment artifact (ZIP file or S3 bundle)."

- symptoms: "Deployment failed at BeforeInstall hook"
  diagnosis: "The BeforeInstall hook script returned a non-zero exit code."
  resolution: "Check the hook script for errors. Review /opt/codedeploy-agent/deployment-root/<deployment-group>/<deployment-id>/logs/scripts.log."

- symptoms: "Deployment stuck — no instances reporting"
  diagnosis: "Instances are not tagged correctly or CodeDeploy agent cannot reach the CodeDeploy service."
  resolution: "Verify instance tags match deployment group filters. Check instance internet/VPC endpoint access to CodeDeploy."

## Output Format

```yaml
root_cause: "Deployment failure — <specific_cause>"
evidence:
  - type: deployment_status
    content: "<deployment details and error>"
  - type: instance_status
    content: "<instance-level lifecycle event details>"
severity: CRITICAL
mitigation:
  immediate: "Fix the identified issue and create a new deployment"
  long_term: "Add deployment validation, implement blue/green for safer deployments"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| If agent not running: install/restart the CodeDeploy agent | YELLOW |
| If appspec.yml missing: add appspec.yml to the deployment artifact | GREEN |
| If lifecycle hook failed: fix the hook script | YELLOW |
| If IAM permission error: update the CodeDeploy service role or instance profile | YELLOW |
| Verify all hook scripts are executable and have correct shebang lines | GREEN |
| Ensure the deployment artifact contains all required files | GREEN |
| Check that the instance profile has permissions for AWS services | YELLOW |
| Use --ignore-application-stop-failures to skip ApplicationStop errors | YELLOW |
| Roll back to the previous revision | RED |

## Escalation Conditions

- Fix requires modifying production pipeline
- Fix requires changing deployment strategy

## Data Sensitivity

- **HIGH**: build logs (may contain secrets), artifact contents
- Deployment artifacts may contain application code, configuration files, and credentials

## Prohibited Actions

- NEVER suggest storing secrets in buildspec.yml
- NEVER suggest skipping approval stages in production pipelines

## Phase 3 — Rollback

1. If a deployment failed, roll back to the last known good revision: `aws codedeploy create-deployment --application-name <app> --deployment-group-name <dg> --revision <previous-revision>`
2. If the CodeDeploy agent was modified, restore the previous agent version
3. If IAM roles were modified, revert to the previous policy version
4. If appspec.yml was changed, revert in source control and redeploy
5. Verify rollback by confirming deployment succeeds and application health checks pass

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

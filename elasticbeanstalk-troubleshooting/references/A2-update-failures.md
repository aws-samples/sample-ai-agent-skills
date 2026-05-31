---
title: "A2 â€” Environment Update Failures"
description: "Diagnose why Elastic Beanstalk environment updates are failing"
status: active
severity: HIGH
triggers:
  - "environment update failed"
  - "update rollback"
  - "configuration update error"
  - "update-environment failed"
owner: devops-agent
objective: "Identify the root cause of environment update failures and restore the environment to a healthy state"
context: "Environment updates can fail due to invalid configuration options, CloudFormation resource update failures, health check failures during rolling updates, or incompatible option settings. The environment may roll back automatically or get stuck in an updating state."
---

## Phase 1 â€” Triage

MUST:
- Check environment status: `aws elasticbeanstalk describe-environments --environment-names <env-name>`
- List recent error events: `aws elasticbeanstalk describe-events --environment-name <env-name> --severity ERROR --max-records 20`
- Check CloudFormation stack status: `aws cloudformation describe-stacks --stack-name awseb-<env-id>-stack`
- Check CloudFormation stack events for failures: `aws cloudformation describe-stack-events --stack-name awseb-<env-id>-stack --max-items 30`
- Get current configuration: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name>`

SHOULD:
- Check environment health: `aws elasticbeanstalk describe-environment-health --environment-name <env-name> --attribute-names All`
- Validate the intended configuration changes: `aws elasticbeanstalk validate-configuration-settings --application-name <app-name> --environment-name <env-name> --option-settings <settings>`
- Check instance health: `aws elasticbeanstalk describe-instances-health --environment-name <env-name>`

MAY:
- Compare with previous configuration versions
- Check CloudTrail for the update event details: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=elasticbeanstalk.amazonaws.com --max-results 10`

## Phase 2 â€” Remediate

MUST:
- If environment is stuck in "Updating", wait for timeout or abort the update
- Fix the invalid configuration option and retry
- If CloudFormation is in UPDATE_ROLLBACK_FAILED, use `aws cloudformation continue-update-rollback`

SHOULD:
- Validate configuration changes before applying to production
- Use immutable update type for safer rollback capability
- Check that option settings are valid for the current platform version

MAY:
- Rebuild the environment if it is in an unrecoverable state
- Create a new environment and perform blue/green swap

## Common Issues

- symptoms: "Environment health transitioned to Degraded during update"
  diagnosis: "New configuration caused health check failures on updated instances."
  resolution: "Review health check path and timeout settings. Ensure application starts within the health check grace period."

- symptoms: "CloudFormation UPDATE_ROLLBACK_FAILED"
  diagnosis: "Rollback itself failed, leaving the stack in an inconsistent state."
  resolution: "Use `continue-update-rollback` with `--resources-to-skip` for resources that cannot be rolled back."

- symptoms: "Invalid option value for namespace"
  diagnosis: "A configuration option value is not valid for the current platform or namespace."
  resolution: "Check `describe-configuration-options` for valid values and constraints."

## Output Format

```yaml
root_cause: "environment_update_failure â€” <specific_cause>"
evidence:
  - type: environment_status
    content: "<describe-environments output>"
  - type: events
    content: "<error events>"
  - type: cloudformation
    content: "<stack events showing update failure>"
severity: HIGH
mitigation:
  immediate: "Fix configuration and retry, or abort and rollback"
  long_term: "Validate configuration changes in staging before production updates"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — Elastic Beanstalk manages full application stacks including compute, networking, and load balancing |
| Data Risk | MEDIUM — Environment changes can affect application availability and data connections |
| Recovery Complexity | MEDIUM — Environments can be rebuilt but require correct configuration and deployment artifacts |
| Blast Radius | HIGH — Environment-level changes affect all instances and the application endpoint |

## Escalation Conditions

- Environment health degraded to Red with customer-facing impact
- Deployment failure with no successful rollback available
- Auto-scaling events failing causing capacity shortage under load
- Platform update causing application incompatibility in production
- CloudFormation stack in UPDATE_ROLLBACK_FAILED state requiring manual intervention
- Load balancer health checks failing across all instances

## Data Sensitivity

- Environment variables may contain database credentials, API keys, and secrets
- Instance profile roles may grant broad AWS service access
- Application source bundles in S3 contain proprietary application code
- Log files may contain PII, authentication tokens, or business data
- .ebextensions may contain infrastructure secrets and custom configurations

## PROHIBITED Actions

- NEVER suggest terminating a production environment without confirming blue-green swap is ready
- NEVER modify environment variables containing secrets without secure handling procedures
- NEVER force a platform update on production without testing on a staging environment first
- NEVER delete application versions that are currently deployed to any environment
- NEVER change the load balancer type on a running environment (requires environment rebuild)
- NEVER disable enhanced health reporting on production environments

## Phase 3 — Rollback

- If deployment failed: use ws elasticbeanstalk abort-environment-update then redeploy last known good version
- If environment configuration change caused issues: apply previous saved configuration via update-environment
- If platform update broke the application: rebuild environment on previous platform version
- If auto-scaling changes caused instability: revert min/max/desired counts to previous values
- If .ebextensions changes caused failures: redeploy previous application version with working extensions
- Document all rollback actions with environment ID, version label, and configuration template name
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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

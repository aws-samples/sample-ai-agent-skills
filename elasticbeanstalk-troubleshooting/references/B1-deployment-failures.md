---
title: "B1 â€” Deployment Failures"
description: "Diagnose why application deployments to Elastic Beanstalk are failing"
status: active
severity: HIGH
triggers:
  - "deployment failed"
  - "deploy error"
  - "application version failed"
  - "deployment timeout"
  - "deploy not working"
owner: devops-agent
objective: "Identify the root cause of deployment failures and successfully deploy the application"
context: "Deployments can fail due to application errors, .ebextensions failures, health check failures during rolling/immutable deployments, S3 source bundle issues, or platform incompatibilities. The deployment method (all-at-once, rolling, immutable) affects failure behavior and rollback."
---

## Phase 1 â€” Triage

MUST:
- Check environment status and last deployment: `aws elasticbeanstalk describe-environments --environment-names <env-name>`
- List recent error events: `aws elasticbeanstalk describe-events --environment-name <env-name> --severity ERROR --max-records 20`
- Check environment health: `aws elasticbeanstalk describe-environment-health --environment-name <env-name> --attribute-names All`
- Check per-instance deployment status: `aws elasticbeanstalk describe-instances-health --environment-name <env-name>`
- Verify the application version exists: `aws elasticbeanstalk describe-application-versions --application-name <app-name> --version-labels <version-label>`

SHOULD:
- Retrieve tail logs for error details: `aws elasticbeanstalk request-environment-info --environment-name <env-name> --info-type tail` then `aws elasticbeanstalk retrieve-environment-info --environment-name <env-name> --info-type tail`
- Check the S3 source bundle is accessible: `aws s3 ls <source-bundle-s3-path>`
- Check deployment configuration: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?Namespace==`aws:elasticbeanstalk:command`]'`

MAY:
- Check CloudFormation events for deployment-related failures
- Review .ebextensions logs in `/var/log/cfn-init.log` via log bundle

## Phase 2 â€” Remediate

MUST:
- Fix the application error or .ebextensions issue identified in logs
- Verify the source bundle is a valid ZIP or WAR file with correct structure
- Redeploy after fixing the issue

SHOULD:
- Test the deployment in a staging environment first
- Use immutable deployment type for production to enable automatic rollback
- Ensure health check path returns 200 within the configured timeout

MAY:
- Roll back to the previous working version: `aws elasticbeanstalk update-environment --environment-name <env-name> --version-label <previous-version>`
- Enable enhanced health reporting for better deployment diagnostics

## Common Issues

- symptoms: "Deployment failed â€” health check failures on new instances"
  diagnosis: "Application fails to start or respond to health checks within the timeout."
  resolution: "Check application startup logs. Increase health check grace period or fix application startup."

- symptoms: ".ebextensions command failed with non-zero exit code"
  diagnosis: "A command or container_command in .ebextensions failed."
  resolution: "Check cfn-init.log for the specific command failure. Fix the command and redeploy."

- symptoms: "Source bundle is not a valid application version"
  diagnosis: "The ZIP/WAR file structure is incorrect or the S3 object is inaccessible."
  resolution: "Verify the bundle structure matches the platform requirements. Check S3 permissions."

- symptoms: "Deployment timeout â€” instances not healthy within deadline"
  diagnosis: "Application takes too long to start or health checks are misconfigured."
  resolution: "Increase the deployment timeout. Check health check path, interval, and threshold settings."

## Output Format

```yaml
root_cause: "deployment_failure â€” <specific_cause>"
evidence:
  - type: environment_health
    content: "<health status and causes>"
  - type: events
    content: "<deployment error events>"
  - type: logs
    content: "<relevant log entries>"
severity: HIGH
mitigation:
  immediate: "Fix the deployment issue and redeploy, or rollback to previous version"
  long_term: "Use immutable deployments and test in staging before production"
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

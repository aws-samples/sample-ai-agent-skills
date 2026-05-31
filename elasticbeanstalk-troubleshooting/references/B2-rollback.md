---
title: "B2 â€” Deployment Rollback Issues"
description: "Diagnose deployment rollback failures and stuck rollback states"
status: active
severity: HIGH
triggers:
  - "rollback failed"
  - "cannot rollback"
  - "stuck in rollback"
  - "previous version not working"
owner: devops-agent
objective: "Successfully rollback to a previous working application version"
context: "Rollback behavior depends on the deployment policy. All-at-once and rolling deployments do not automatically rollback. Immutable and traffic-splitting deployments rollback automatically on failure. Manual rollback deploys a previous application version."
---

## Phase 1 â€” Triage

MUST:
- Check current environment status: `aws elasticbeanstalk describe-environments --environment-names <env-name>`
- List application versions: `aws elasticbeanstalk describe-application-versions --application-name <app-name> --max-records 10`
- Check current deployed version: `aws elasticbeanstalk describe-environments --environment-names <env-name> --query 'Environments[0].VersionLabel'`
- List recent events: `aws elasticbeanstalk describe-events --environment-name <env-name> --max-records 20`

SHOULD:
- Check deployment policy: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?Namespace==`aws:elasticbeanstalk:command`]'`
- Check environment health: `aws elasticbeanstalk describe-environment-health --environment-name <env-name> --attribute-names All`

MAY:
- Check CloudFormation stack for rollback state: `aws cloudformation describe-stacks --stack-name awseb-<env-id>-stack`

## Phase 2 â€” Remediate

MUST:
- Deploy the previous working version: `aws elasticbeanstalk update-environment --environment-name <env-name> --version-label <previous-version>`
- Monitor the rollback deployment for health check success
- Verify the environment returns to healthy state after rollback

SHOULD:
- Switch to immutable deployment policy for automatic rollback capability
- Keep previous application versions available (do not delete them)
- Test new versions in staging before production deployment

MAY:
- Use blue/green deployment for zero-downtime rollback via CNAME swap
- Rebuild the environment if rollback deployment also fails

## Common Issues

- symptoms: "Manual rollback deployment also fails"
  diagnosis: "The issue is environmental (configuration, infrastructure) not application-specific."
  resolution: "Check environment configuration, instance health, and infrastructure. Fix the underlying issue."

- symptoms: "Previous application version no longer exists in S3"
  diagnosis: "The source bundle was deleted from the S3 bucket."
  resolution: "Re-upload the application bundle and create a new application version, then deploy."

- symptoms: "Immutable deployment did not rollback automatically"
  diagnosis: "The health check passed on the new instances despite application issues."
  resolution: "Review health check configuration. Ensure it validates actual application functionality."

## Output Format

```yaml
root_cause: "rollback_issue â€” <specific_cause>"
evidence:
  - type: environment_status
    content: "<current status and version>"
  - type: versions
    content: "<available application versions>"
  - type: events
    content: "<rollback-related events>"
severity: HIGH
mitigation:
  immediate: "Deploy the last known working version"
  long_term: "Use immutable deployments and maintain application version history"
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

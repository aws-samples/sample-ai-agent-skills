---
title: "C2 â€” Health Check Configuration Issues"
description: "Diagnose health check misconfiguration causing false positives or negatives"
status: active
severity: MEDIUM
triggers:
  - "health check failing"
  - "health check path"
  - "health check timeout"
  - "false unhealthy"
  - "health check configuration"
owner: devops-agent
objective: "Configure health checks correctly to accurately reflect application health"
context: "Elastic Beanstalk uses ELB health checks (basic) and/or the health agent (enhanced). Misconfigured health check paths, timeouts, or thresholds cause false positives (unhealthy when healthy) or false negatives (healthy when unhealthy). The health check path must return HTTP 200 within the timeout."
---

## Phase 1 â€” Triage

MUST:
- Check health check configuration: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?Namespace==`aws:elasticbeanstalk:application`]'`
- Check ELB health check settings: `aws elbv2 describe-target-groups --target-group-arns <tg-arn>`
- Check target health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- Check enhanced health configuration: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?Namespace==`aws:elasticbeanstalk:healthreporting:system`]'`

SHOULD:
- Verify the health check path exists in the application and returns 200
- Check health check interval, timeout, and threshold values
- Review instance health details: `aws elasticbeanstalk describe-instances-health --environment-name <env-name>`

MAY:
- Test the health check endpoint directly from an instance
- Check if the application requires authentication on the health check path

## Phase 2 â€” Remediate

MUST:
- Set the health check path to a valid endpoint that returns HTTP 200
- Ensure the health check timeout is less than the interval
- Set appropriate healthy/unhealthy thresholds for the application

SHOULD:
- Create a dedicated health check endpoint (e.g., `/health`) that checks application dependencies
- Set the health check grace period to accommodate application startup time
- Use enhanced health reporting for more accurate health assessment

MAY:
- Configure different health check paths for different deployment stages
- Implement deep health checks that verify database and cache connectivity

## Common Issues

- symptoms: "New instances always fail health checks during deployment"
  diagnosis: "Application startup time exceeds the health check grace period."
  resolution: "Increase HealthCheckGracePeriod in the ASG configuration or speed up application startup."

- symptoms: "Health check passes but application is not functioning correctly"
  diagnosis: "Health check path returns 200 without validating actual application functionality."
  resolution: "Implement a health check endpoint that verifies critical dependencies."

- symptoms: "Health check path returns 301 redirect"
  diagnosis: "The health check path redirects (e.g., HTTP to HTTPS) which ELB treats as unhealthy."
  resolution: "Use a path that returns 200 directly without redirects."

## Output Format

```yaml
root_cause: "health_check_misconfiguration â€” <specific_cause>"
evidence:
  - type: health_check_config
    content: "<health check path, interval, timeout, thresholds>"
  - type: target_health
    content: "<ELB target health status>"
  - type: instance_health
    content: "<enhanced health details>"
severity: MEDIUM
mitigation:
  immediate: "Fix health check configuration to match application behavior"
  long_term: "Implement dedicated health check endpoints with dependency validation"
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

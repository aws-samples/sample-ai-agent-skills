---
title: "C3 â€” Enhanced Health Reporting Issues"
description: "Diagnose problems with enhanced health reporting and the health agent"
status: active
severity: MEDIUM
triggers:
  - "enhanced health not working"
  - "health agent"
  - "basic health only"
  - "no health metrics"
  - "health reporting"
owner: devops-agent
objective: "Enable and configure enhanced health reporting for accurate environment monitoring"
context: "Enhanced health reporting provides application-level metrics (request latency, HTTP status codes, instance health details) via the Beanstalk health agent. It requires a supported platform version and explicit enablement. Without it, only basic ELB-based health is available."
---

## Phase 1 â€” Triage

MUST:
- Check health reporting type: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?Namespace==`aws:elasticbeanstalk:healthreporting:system`]'`
- Check platform version supports enhanced health: `aws elasticbeanstalk describe-platform-version --platform-arn <platform-arn>`
- Check environment health: `aws elasticbeanstalk describe-environment-health --environment-name <env-name> --attribute-names All`
- Check instance health details: `aws elasticbeanstalk describe-instances-health --environment-name <env-name>`

SHOULD:
- Verify the health agent is running on instances (check `/var/log/healthd/` logs)
- Check the instance profile has CloudWatch permissions for health metric publishing
- Verify the environment is using Amazon Linux 2 or later platform

MAY:
- Check CloudWatch for health metrics: `aws cloudwatch list-metrics --namespace AWS/ElasticBeanstalk --dimensions Name=EnvironmentName,Value=<env-name>`
- Review health agent configuration files on instances

## Phase 2 â€” Remediate

MUST:
- Enable enhanced health: set `aws:elasticbeanstalk:healthreporting:system` SystemType to `enhanced`
- Ensure the platform version supports enhanced health (Amazon Linux 2+)
- Verify the instance profile includes CloudWatch permissions

SHOULD:
- Configure health check URL for application-level health validation
- Enable CloudWatch custom metrics publishing for enhanced health data
- Set appropriate health transition thresholds

MAY:
- Upgrade to a newer platform version if current version lacks health agent support
- Configure health-based rolling updates to use enhanced health for deployment decisions

## Common Issues

- symptoms: "Enhanced health shows 'Unknown' for all instances"
  diagnosis: "Health agent is not running or cannot communicate with the Beanstalk service."
  resolution: "Verify the health agent is installed and running. Check instance profile permissions."

- symptoms: "Only basic health available â€” no request metrics"
  diagnosis: "Enhanced health is not enabled or the platform does not support it."
  resolution: "Set SystemType to 'enhanced' in the healthreporting:system namespace."

- symptoms: "Health metrics not appearing in CloudWatch"
  diagnosis: "CloudWatch custom metrics publishing is not enabled."
  resolution: "Enable ConfigDocument with CloudWatchMetrics in the healthreporting:system namespace."

## Output Format

```yaml
root_cause: "enhanced_health_issue â€” <specific_cause>"
evidence:
  - type: health_config
    content: "<health reporting configuration>"
  - type: platform
    content: "<platform version and health agent support>"
  - type: instance_health
    content: "<instance-level health details>"
severity: MEDIUM
mitigation:
  immediate: "Enable enhanced health and verify health agent is running"
  long_term: "Upgrade to a platform version with full enhanced health support"
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

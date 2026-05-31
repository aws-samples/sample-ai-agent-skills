---
title: "H2 â€” CloudWatch Integration Issues"
description: "Diagnose CloudWatch Logs and metrics integration problems"
status: active
severity: LOW
triggers:
  - "CloudWatch logs not appearing"
  - "metrics missing"
  - "log streaming"
  - "CloudWatch integration"
  - "custom metrics"
owner: devops-agent
objective: "Configure CloudWatch Logs and metrics integration for the Beanstalk environment"
context: "Beanstalk can stream instance logs to CloudWatch Logs and publish custom metrics. This requires enabling log streaming in the environment configuration, proper IAM permissions, and the CloudWatch Logs agent on instances. Enhanced health can also publish metrics to CloudWatch."
---

## Phase 1 â€” Triage

MUST:
- Check CloudWatch Logs streaming config: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?Namespace==`aws:elasticbeanstalk:cloudwatch:logs`]'`
- Check for log groups: `aws logs describe-log-groups --log-group-name-prefix /aws/elasticbeanstalk/<env-name>`
- Check enhanced health metrics config: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?Namespace==`aws:elasticbeanstalk:healthreporting:system`]'`
- Verify instance profile has CloudWatch permissions: `aws iam list-attached-role-policies --role-name <instance-role>`

SHOULD:
- Check log streams for recent entries: `aws logs describe-log-streams --log-group-name /aws/elasticbeanstalk/<env-name>/<log-name> --order-by LastEventTime --descending --max-items 5`
- Verify CloudWatch Logs agent is running on instances
- Check for CloudWatch metrics: `aws cloudwatch list-metrics --namespace AWS/ElasticBeanstalk --dimensions Name=EnvironmentName,Value=<env-name>`

MAY:
- Check CloudWatch Logs agent configuration on instances
- Review log retention settings: `aws logs describe-log-groups --log-group-name-prefix /aws/elasticbeanstalk/<env-name>`

## Phase 2 â€” Remediate

MUST:
- Enable log streaming: set `StreamLogs` to `true` in `aws:elasticbeanstalk:cloudwatch:logs`
- Add CloudWatch Logs permissions to the instance profile (logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents)
- Set log retention period to avoid unbounded log storage costs

SHOULD:
- Enable enhanced health CloudWatch metrics publishing
- Configure custom log paths for application-specific logs
- Set up CloudWatch alarms for key metrics

MAY:
- Use CloudWatch Logs Insights for log analysis
- Configure metric filters to create custom metrics from log patterns

## Common Issues

- symptoms: "Log groups created but no log streams or events"
  diagnosis: "CloudWatch Logs agent is not running or cannot write to the log group."
  resolution: "Check agent status on instances. Verify IAM permissions include logs:PutLogEvents."

- symptoms: "Enhanced health metrics not appearing in CloudWatch"
  diagnosis: "CloudWatch custom metrics publishing is not enabled in the health reporting config."
  resolution: "Enable ConfigDocument with CloudWatchMetrics in the healthreporting:system namespace."

- symptoms: "Log groups accumulating â€” high CloudWatch Logs costs"
  diagnosis: "Log retention is set to 'Never expire' (default)."
  resolution: "Set RetentionInDays in the cloudwatch:logs namespace (e.g., 30 or 90 days)."

## Output Format

```yaml
root_cause: "cloudwatch_integration_issue â€” <specific_cause>"
evidence:
  - type: log_config
    content: "<CloudWatch Logs streaming configuration>"
  - type: log_groups
    content: "<log group and stream status>"
  - type: permissions
    content: "<IAM permissions for CloudWatch>"
severity: LOW
mitigation:
  immediate: "Enable log streaming and fix permissions"
  long_term: "Configure log retention and set up CloudWatch alarms"
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

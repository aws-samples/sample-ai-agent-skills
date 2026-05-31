---
title: "E1 â€” Auto Scaling Issues"
description: "Diagnose auto-scaling configuration and scaling behavior problems"
status: active
severity: MEDIUM
triggers:
  - "auto scaling not working"
  - "instances not scaling"
  - "scaling policy"
  - "min max instances"
  - "scaling trigger"
owner: devops-agent
objective: "Configure auto-scaling to correctly respond to load changes"
context: "Elastic Beanstalk manages an Auto Scaling group with configurable triggers (CPU, network, latency, etc.), min/max instance counts, and cooldown periods. Scaling issues can be caused by misconfigured triggers, incorrect thresholds, cooldown periods, or instance launch failures."
---

## Phase 1 â€” Triage

MUST:
- Check ASG configuration: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name>`
- Check scaling policies: `aws autoscaling describe-policies --auto-scaling-group-name <asg-name>`
- Check scaling activities: `aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg-name> --max-items 20`
- Check Beanstalk scaling config: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?Namespace==`aws:autoscaling:asg` || Namespace==`aws:autoscaling:trigger`]'`

SHOULD:
- Check CloudWatch alarms for scaling triggers: `aws cloudwatch describe-alarms --alarm-name-prefix awseb-<env-id>`
- Check instance launch failures in ASG activities
- Verify instance type availability in the configured subnets/AZs

MAY:
- Check EC2 service quotas: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A`
- Review CloudWatch metrics for the scaling trigger metric

## Phase 2 â€” Remediate

MUST:
- Fix scaling trigger thresholds to match actual application load patterns
- Ensure min/max instance counts are appropriate for the workload
- Verify instance types are available in the configured AZs

SHOULD:
- Set appropriate cooldown periods to prevent scaling thrashing
- Use multiple AZs for high availability
- Configure scaling based on the most relevant metric (CPU, request count, latency)

MAY:
- Use predictive scaling for predictable traffic patterns
- Configure scheduled scaling for known traffic peaks

## Common Issues

- symptoms: "Instances not scaling up despite high CPU"
  diagnosis: "Scaling trigger threshold is too high or cooldown period prevents scaling."
  resolution: "Lower the UpperThreshold or reduce the cooldown period."

- symptoms: "Scaling activity shows 'Failed to launch instances'"
  diagnosis: "Instance type not available in the AZ or EC2 limit reached."
  resolution: "Use multiple instance types or request a quota increase."

- symptoms: "Instances scale up and down repeatedly (thrashing)"
  diagnosis: "Cooldown period is too short or thresholds are too close together."
  resolution: "Increase cooldown period and widen the gap between upper and lower thresholds."

## Output Format

```yaml
root_cause: "auto_scaling_issue â€” <specific_cause>"
evidence:
  - type: asg_config
    content: "<ASG min, max, desired, and scaling policies>"
  - type: scaling_activities
    content: "<recent scaling activities and outcomes>"
  - type: cloudwatch_alarms
    content: "<scaling trigger alarm states>"
severity: MEDIUM
mitigation:
  immediate: "Adjust scaling triggers and thresholds"
  long_term: "Monitor scaling patterns and optimize trigger configuration"
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

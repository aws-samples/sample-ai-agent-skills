---
title: "C1 — Red/Yellow Environment Health"
description: "Diagnose and resolve red or yellow environment health status"
status: active
severity: HIGH
triggers:
  - "red health"
  - "yellow health"
  - "severe health"
  - "degraded health"
  - "environment unhealthy"
owner: devops-agent
objective: "Identify the cause of degraded health and restore the environment to green/OK status"
context: "Elastic Beanstalk health status reflects instance health, application health, and system health. Red (Severe) means most instances are failing. Yellow (Warning/Degraded) means some instances or requests are failing. Health causes provide specific reasons for the status."
---

## Phase 1 — Triage

MUST:
- Check environment health with causes: `aws elasticbeanstalk describe-environment-health --environment-name <env-name> --attribute-names All`
- Check per-instance health: `aws elasticbeanstalk describe-instances-health --environment-name <env-name>`
- List recent error events: `aws elasticbeanstalk describe-events --environment-name <env-name> --severity ERROR --max-records 20`
- Check ELB target health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- Check instance status: `aws ec2 describe-instance-status --instance-ids <instance-ids>`

SHOULD:
- Retrieve tail logs: `aws elasticbeanstalk request-environment-info --environment-name <env-name> --info-type tail` then `aws elasticbeanstalk retrieve-environment-info --environment-name <env-name> --info-type tail`
- Check CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=AutoScalingGroupName,Value=<asg-name> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check ASG instance count vs desired: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name>`

MAY:
- Check for recent deployments that may have caused the health change
- Review application error logs for 5xx responses

## Phase 2 — Remediate

MUST:
- Address the specific health cause reported by enhanced health
- Fix application errors causing 5xx responses
- Ensure instances pass both ELB health checks and enhanced health checks

SHOULD:
- Restart unhealthy instances: `aws elasticbeanstalk restart-app-server --environment-name <env-name>`
- Roll back recent deployments if they caused the health degradation
- Adjust health check thresholds if the application has slow startup

MAY:
- Scale up temporarily to handle traffic while fixing unhealthy instances
- Rebuild the environment if health cannot be restored

## Common Issues

- symptoms: "5xx responses exceeding threshold — health turned red"
  diagnosis: "Application is returning HTTP 500 errors to a significant percentage of requests."
  resolution: "Check application logs for the error cause. Fix the application bug and redeploy."

- symptoms: "Instances failing ELB health checks"
  diagnosis: "Application is not responding on the health check path within the timeout."
  resolution: "Verify health check path returns 200. Check application is listening on the correct port."

- symptoms: "CPU utilization above threshold — health degraded"
  diagnosis: "Instances are overloaded. Application or traffic spike causing high CPU."
  resolution: "Scale out by increasing instance count. Optimize application performance."

- symptoms: "Instances terminated by Auto Scaling — health fluctuating"
  diagnosis: "ASG is replacing unhealthy instances but replacements also fail health checks."
  resolution: "Fix the underlying application or configuration issue causing health check failures."

## Output Format

```yaml
root_cause: "health_degradation — <specific_cause>"
evidence:
  - type: environment_health
    content: "<health status, color, and causes>"
  - type: instance_health
    content: "<per-instance health details>"
  - type: elb_health
    content: "<target group health check results>"
severity: HIGH
mitigation:
  immediate: "Fix the application or configuration issue causing health failures"
  long_term: "Set up CloudWatch alarms for health status changes"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH � Elastic Beanstalk manages full application stacks including compute, networking, and load balancing |
| Data Risk | MEDIUM � Environment changes can affect application availability and data connections |
| Recovery Complexity | MEDIUM � Environments can be rebuilt but require correct configuration and deployment artifacts |
| Blast Radius | HIGH � Environment-level changes affect all instances and the application endpoint |

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

## Phase 3 � Rollback

- If deployment failed: use ws elasticbeanstalk abort-environment-update then redeploy last known good version
- If environment configuration change caused issues: apply previous saved configuration via update-environment
- If platform update broke the application: rebuild environment on previous platform version
- If auto-scaling changes caused instability: revert min/max/desired counts to previous values
- If .ebextensions changes caused failures: redeploy previous application version with working extensions
- Document all rollback actions with environment ID, version label, and configuration template name
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

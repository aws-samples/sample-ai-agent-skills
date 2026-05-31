---
title: "C1 — Health Check Failures"
description: "Diagnose health check failures causing unexpected instance terminations"
status: active
severity: HIGH
triggers:
  - "health check failed"
  - "instance terminated unexpectedly"
  - "unhealthy instance"
  - "health check replacing instances"
  - "instances cycling"
owner: devops-agent
objective: "Identify why health checks are failing and stop unnecessary terminations"
context: "Health check failures can be caused by: EC2 status check failures, ELB health check failures, application not ready within grace period, incorrect health check path/port, security group blocking health checks, or application crashes."
---

## Phase 1 — Triage

MUST:
- Check health check type and grace period: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].{HealthCheckType:HealthCheckType,GracePeriod:HealthCheckGracePeriod}'`
- Check instance health states: `aws autoscaling describe-auto-scaling-instances --query 'AutoScalingInstances[?AutoScalingGroupName==`<asg-name>`].{Id:InstanceId,Health:HealthStatus,State:LifecycleState}'`
- Check scaling activities for health check terminations: `aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg-name> --query 'Activities[?contains(Cause,`health check`)].{Time:StartTime,Cause:Cause}'`
- If ELB type, check target health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`

SHOULD:
- Check EC2 status checks: `aws ec2 describe-instance-status --instance-ids <instance-id>`
- Verify health check configuration on target group: `aws elbv2 describe-target-groups --target-group-arns <tg-arn> --query 'TargetGroups[*].{Path:HealthCheckPath,Port:HealthCheckPort,Interval:HealthCheckIntervalSeconds,Threshold:UnhealthyThresholdCount}'`
- Check if grace period is sufficient for application startup

MAY:
- Review application logs on terminated instances (if available)
- Check if security groups allow health check traffic

## Phase 2 — Remediate

MUST:
- If grace period too short: increase it to cover application startup time
- If ELB health check path wrong: fix the health check path/port
- If security group blocking: allow health check traffic
- If application crashing: fix the application issue

SHOULD:
- Increase grace period: `aws autoscaling update-auto-scaling-group --auto-scaling-group-name <asg-name> --health-check-grace-period 600`
- Adjust target group health check settings for slower applications

MAY:
- Implement a dedicated health check endpoint in the application
- Use startup probes before enabling health checks

## Output Format

```yaml
root_cause: "health_check_failure — <specific_cause>"
evidence:
  - type: health_check_config
    content: "<health check type and grace period>"
  - type: target_health
    content: "<target group health status>"
severity: HIGH
mitigation:
  immediate: "Fix health check configuration or application issue"
  long_term: "Implement proper health check endpoints and grace periods"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Increase health check grace period | YELLOW | Delays unhealthy detection — may mask real issues |
| Fix health check path/port | YELLOW | Changes health check behavior |
| Allow health check traffic in security groups | YELLOW | Network access change |
| Fix application issue | YELLOW | Application change — test before production |
| Adjust target group health check settings | YELLOW | Changes health check sensitivity |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Health check changes may mask real application failures
- Security group changes affect network access patterns

## Data Sensitivity

- Health check configuration reveals application endpoints and ports
- Health check failures may indicate application vulnerabilities
- Instance termination patterns reveal application stability

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER set health check grace period excessively high to hide real failures
- NEVER disable health checks entirely in production

## Phase 3 — Rollback

- If grace period was increased too much: reduce to appropriate value
- If health check path was changed: restore original health check configuration
- If security group was modified: revert security group rules
- If target group health check was adjusted: restore original settings
- Document all health check changes for operational audit trail

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

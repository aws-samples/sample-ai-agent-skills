---
title: "A1 — Target Health Failures"
description: "Diagnose and resolve ELB target health check failures"
status: active
severity: CRITICAL
triggers:
  - "unhealthy target"
  - "health check failed"
  - "target draining"
  - "target unused"
owner: devops-agent
objective: "Identify why targets are failing health checks and restore them to healthy state"
context: "ELB health checks verify that registered targets can receive traffic. Targets can be unhealthy due to application failures, misconfigured health check paths, security group issues, or network connectivity problems. ALB supports HTTP/HTTPS health checks; NLB supports TCP, HTTP, and HTTPS. Unhealthy targets are removed from the load balancer's rotation, potentially causing service degradation."
---

## Phase 1 — Triage

MUST:
- Check target health status: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- Review health check configuration: `aws elbv2 describe-target-groups --target-group-arns <tg-arn> --query 'TargetGroups[0].{Path:HealthCheckPath,Port:HealthCheckPort,Protocol:HealthCheckProtocol,Interval:HealthCheckIntervalSeconds,Threshold:HealthyThresholdCount,UnhealthyThreshold:UnhealthyThresholdCount}'`
- Check the target's security group allows health check traffic: `aws ec2 describe-security-groups --group-ids <target-sg-id>`
- Verify the target instance/container is running: `aws ec2 describe-instance-status --instance-ids <instance-id>` or check ECS task status
- Test the health check endpoint directly from the target: `curl -v http://localhost:<port><health-check-path>`

SHOULD:
- Check if the health check port differs from the traffic port
- Verify the application is listening on the expected port: `ss -tlnp | grep <port>` (on the target)
- Review target group attributes for deregistration delay: `aws elbv2 describe-target-group-attributes --target-group-arn <tg-arn>`
- Check CloudWatch metrics for UnHealthyHostCount: `aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name UnHealthyHostCount --dimensions Name=TargetGroup,Value=<tg-arn-suffix>`

MAY:
- Check VPC flow logs for denied traffic between ELB and targets
- Review application logs on the target for errors during health checks
- Verify NACLs allow traffic on the health check port

## Phase 2 — Remediate

MUST:
- If application not responding: restart the application or service on the target
- If security group blocks health checks: add inbound rule allowing traffic from the ELB security group (ALB) or VPC CIDR (NLB) on the health check port
- If health check path returns non-200: fix the application endpoint or update the health check path/matcher
- If health check port is wrong: update the target group health check port: `aws elbv2 modify-target-group --target-group-arn <tg-arn> --health-check-port <correct-port>`

SHOULD:
- Adjust health check thresholds if targets are flapping: increase unhealthy threshold or interval
- Verify targets return to healthy state after remediation: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- Check if the issue affects all targets or specific ones

MAY:
- Implement a dedicated lightweight health check endpoint in the application
- Set up CloudWatch alarms for UnHealthyHostCount
- Review and optimize health check intervals for the workload

## Common Issues

- symptoms: "All targets show 'Health checks failed with these codes: [502]'"
  diagnosis: "Application is returning 502 errors — likely crashed or not started."
  resolution: "Check application status on targets. Restart the application. Review application logs."

- symptoms: "Targets show 'Elb.InitialHealthChecking' for extended time"
  diagnosis: "New targets need to pass the healthy threshold count before being marked healthy."
  resolution: "Wait for the health check interval × healthy threshold. Reduce threshold for faster registration."

- symptoms: "Targets healthy on one port but unhealthy on health check port"
  diagnosis: "Health check port differs from traffic port and the application is not listening on it."
  resolution: "Update health check to use the traffic port, or ensure the application listens on the health check port."

- symptoms: "Targets intermittently flip between healthy and unhealthy"
  diagnosis: "Application is slow to respond, causing health check timeouts."
  resolution: "Increase health check timeout. Optimize the health check endpoint response time."

## Output Format

```yaml
root_cause: "Target health failure — <specific_cause>"
evidence:
  - type: target_health
    content: "<describe-target-health output>"
  - type: health_check_config
    content: "<health check configuration details>"
severity: CRITICAL
mitigation:
  immediate: "Fix application, security group, or health check configuration"
  long_term: "Implement robust health check endpoints and monitoring"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get): GREEN — read-only"
  - "Phase 2 health check configuration changes: YELLOW — may affect traffic routing"
  - "Phase 2 security group modifications: YELLOW — state-changing but recoverable"

## Escalation Conditions

escalation_conditions:
  - "All targets in a target group are unhealthy (complete service outage)"
  - "Health check changes could affect production traffic routing"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Issue involves cross-AZ target health failures"
  - "Security group changes affect shared resources"

## Data Sensitivity

data_sensitivity:
  - command: "describe-target-health"
    sensitivity: LOW
    contains: "Target IDs and health status"
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Network access rules and IP ranges"
  - command: "application health check response"
    sensitivity: MEDIUM
    contains: "Application status information"

## Prohibited Actions

prohibited_actions:
  - "NEVER disable health checks to mask unhealthy targets"
  - "NEVER set health check interval below 5 seconds without understanding the load impact"
  - "NEVER open security groups to 0.0.0.0/0 for health check troubleshooting"
  - "NEVER deregister all targets simultaneously from a production target group"

## Phase 3 — Rollback

- Before modifying health check settings, document current configuration: `aws elbv2 describe-target-groups --target-group-arns <tg-arn>`
- If health check changes cause all targets to become unhealthy, revert to previous settings
- If security group changes break other services, restore previous rules
- Verify target health returns to the expected state after rollback
- Monitor CloudWatch UnHealthyHostCount metric after changes

---
title: "F1 — ELB Integration Issues"
description: "Diagnose Auto Scaling integration problems with Elastic Load Balancing"
status: active
severity: HIGH
triggers:
  - "ELB not registering instances"
  - "target group empty"
  - "instances not in target group"
  - "ELB health check mismatch"
  - "deregistration delay"
owner: devops-agent
objective: "Identify and resolve ELB integration issues with Auto Scaling"
context: "Auto Scaling integrates with ALB/NLB target groups to automatically register/deregister instances. Issues include: target group not attached, instances failing ELB health checks, deregistration delay too long/short, or health check type mismatch."
---

## Phase 1 — Triage

MUST:
- Check attached target groups: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].TargetGroupARNs'`
- Check target health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- Check ASG health check type: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].HealthCheckType'`
- Check target group health check config: `aws elbv2 describe-target-groups --target-group-arns <tg-arn>`

SHOULD:
- Verify security groups allow traffic from load balancer to instances
- Check deregistration delay: `aws elbv2 describe-target-group-attributes --target-group-arn <tg-arn> --query 'Attributes[?Key==`deregistration_delay.timeout_seconds`]'`
- Verify instances are in correct subnets for the target group

MAY:
- Check load balancer listener configuration
- Review connection draining behavior

## Phase 2 — Remediate

MUST:
- If target group not attached: `aws autoscaling attach-load-balancer-target-groups --auto-scaling-group-name <asg-name> --target-group-arns <tg-arn>`
- If health check type is EC2: switch to ELB for proper integration
- Fix security groups to allow LB-to-instance traffic

SHOULD:
- Set health check type to ELB: `aws autoscaling update-auto-scaling-group --auto-scaling-group-name <asg-name> --health-check-type ELB --health-check-grace-period 300`
- Adjust deregistration delay for graceful shutdown

MAY:
- Implement connection draining in the application
- Use slow start mode on target group for gradual traffic ramp-up

## Output Format

```yaml
root_cause: "elb_integration_issue — <specific_cause>"
evidence:
  - type: target_groups
    content: "<attached target groups>"
  - type: target_health
    content: "<instance health in target group>"
severity: HIGH
mitigation:
  immediate: "Fix target group attachment or health check configuration"
  long_term: "Implement proper ELB health checks with appropriate grace period"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Attach target group to ASG | YELLOW | Registers instances with load balancer — starts receiving traffic |
| Switch health check type to ELB | YELLOW | Changes health check behavior — may terminate unhealthy instances |
| Fix security groups for LB-to-instance traffic | YELLOW | Network access change |
| Adjust deregistration delay | GREEN | Changes graceful shutdown timing |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Target group attachment immediately registers instances for traffic
- Health check type change may trigger instance replacements

## Data Sensitivity

- Target group configuration reveals load balancing architecture
- Health check paths reveal application endpoints
- Security group rules reveal network access patterns

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER detach target groups from production ASGs without traffic migration
- NEVER set deregistration delay to 0 in production (causes connection drops)

## Phase 3 — Rollback

- If target group was attached incorrectly: detach using `aws autoscaling detach-load-balancer-target-groups`
- If health check type was changed: revert to previous type
- If security groups were modified: restore original rules
- Document all ELB integration changes for operational audit trail

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

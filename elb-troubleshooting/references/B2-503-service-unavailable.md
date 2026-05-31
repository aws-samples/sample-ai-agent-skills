---
title: "B2 — 503 Service Unavailable"
description: "Diagnose and resolve ELB 503 Service Unavailable errors"
status: active
severity: CRITICAL
triggers:
  - "503 service unavailable"
  - "HTTP 503"
  - "ELB 503"
  - "no healthy targets"
owner: devops-agent
objective: "Identify why the load balancer is returning 503 errors and restore service availability"
context: "A 503 Service Unavailable from an ALB indicates no healthy targets are available to handle the request, or the load balancer itself is not ready. This is distinct from 502 (bad response from target). Common causes: all targets unhealthy, no targets registered, target group not associated with a listener rule, or ALB in provisioning state. For CLB, 503 can also indicate the load balancer is at capacity."
---

## Phase 1 — Triage

MUST:
- Check target health for all target groups: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- Verify targets are registered: check if the target list is empty
- Check load balancer state: `aws elbv2 describe-load-balancers --load-balancer-arns <lb-arn> --query 'LoadBalancers[0].State'`
- Verify listener rules route to the correct target group: `aws elbv2 describe-rules --listener-arn <listener-arn>`
- Check CloudWatch HealthyHostCount: `aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name HealthyHostCount --dimensions Name=TargetGroup,Value=<tg-arn-suffix> --start-time <time> --end-time <time> --period 60 --statistics Minimum`

SHOULD:
- Check if the ALB has listeners configured: `aws elbv2 describe-listeners --load-balancer-arn <lb-arn>`
- Verify the default action on the listener is not returning a fixed 503 response
- Check if Auto Scaling is maintaining the desired capacity
- Review recent deployment or scaling events

MAY:
- Check ALB access logs for patterns in 503 responses
- Verify the ALB subnets have available IP addresses
- Check if WAF rules are blocking all traffic (returning 503)

## Phase 2 — Remediate

MUST:
- If no healthy targets: investigate and fix target health (see A1 runbook)
- If no targets registered: register targets or fix Auto Scaling configuration
- If listener rule misconfigured: update the rule to route to the correct target group
- If ALB in provisioning state: wait for it to become active

SHOULD:
- Scale up the target group to add more healthy targets
- Verify the fix by checking HealthyHostCount returns to expected levels
- Review and fix the root cause of target unhealthiness

MAY:
- Implement a custom error page for 503 responses using ALB fixed-response actions
- Set up CloudWatch alarms for HealthyHostCount dropping to 0
- Configure Auto Scaling policies to maintain minimum healthy targets

## Common Issues

- symptoms: "503 for all requests — HealthyHostCount is 0"
  diagnosis: "All targets are unhealthy or no targets are registered."
  resolution: "Fix target health issues (see A1 runbook) or register new targets."

- symptoms: "503 for specific paths only"
  diagnosis: "Listener rule for that path routes to a target group with no healthy targets."
  resolution: "Check the path-based routing rules and the associated target group's health."

- symptoms: "503 after ALB creation"
  diagnosis: "ALB is still provisioning or no listener/target group is configured."
  resolution: "Wait for ALB to become active. Verify listeners and target groups are configured."

- symptoms: "Intermittent 503 during scaling events"
  diagnosis: "New targets not yet healthy while old targets are being deregistered."
  resolution: "Increase minimum healthy targets. Use slow-start mode on the target group."

## Output Format

```yaml
root_cause: "503 Service Unavailable — <specific_cause>"
evidence:
  - type: target_health
    content: "<describe-target-health output>"
  - type: healthy_host_count
    content: "<CloudWatch HealthyHostCount metric>"
severity: CRITICAL
mitigation:
  immediate: "Restore healthy targets or fix routing configuration"
  long_term: "Implement auto-scaling, health monitoring, and minimum capacity policies"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/metrics): GREEN — read-only"
  - "Phase 2 listener rule changes: YELLOW — affects traffic routing"
  - "Phase 2 target registration: YELLOW — adds targets to receive production traffic"

## Escalation Conditions

escalation_conditions:
  - "Complete service outage — no healthy targets across all AZs"
  - "503 errors persist after target health remediation"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Issue requires ALB recreation or major reconfiguration"
  - "Auto Scaling is unable to launch new instances"

## Data Sensitivity

data_sensitivity:
  - command: "describe-target-health"
    sensitivity: LOW
    contains: "Target IDs and health status"
  - command: "describe-rules"
    sensitivity: MEDIUM
    contains: "Routing rules and target group associations"
  - command: "ALB access logs"
    sensitivity: MEDIUM
    contains: "Client IPs and request details"

## Prohibited Actions

prohibited_actions:
  - "NEVER delete a production load balancer to resolve 503 errors"
  - "NEVER remove listener rules without understanding the routing impact"
  - "NEVER register unhealthy targets just to clear 503 errors"
  - "NEVER disable health checks to force targets into healthy state"

## Phase 3 — Rollback

- Before modifying listener rules, document current configuration: `aws elbv2 describe-rules --listener-arn <listener-arn>`
- If listener rule changes cause routing issues, restore previous rules
- If target registration changes worsen the situation, deregister the problematic targets
- If Auto Scaling changes were made, verify the ASG configuration is correct
- Monitor 503 error rate and HealthyHostCount after any changes

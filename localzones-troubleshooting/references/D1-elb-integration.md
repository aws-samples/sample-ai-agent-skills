---
title: "D1 — ELB Integration"
description: "Diagnose Elastic Load Balancing integration issues with Local Zones"
status: active
severity: HIGH
triggers: ["ELB not working in Local Zone", "ALB Local Zone error", "NLB Local Zone", "load balancer subnet error"]
owner: devops-agent
objective: "Resolve ELB integration issues with Local Zone subnets"
context: "ELB can be configured with Local Zone subnets to distribute traffic to instances in Local Zones. Not all ELB types may be available in every Local Zone. Cross-zone load balancing behavior differs."
---

## Phase 1 — Triage

MUST:
- Check ELB subnet configuration: `aws elbv2 describe-load-balancers --names <lb-name> --query 'LoadBalancers[0].AvailabilityZones'`
- Verify Local Zone subnet is included in ELB: `aws elbv2 describe-load-balancers --names <lb-name>`
- Check target group health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`

SHOULD:
- Verify ELB type (ALB/NLB) is supported in the Local Zone
- Check security group allows traffic from ELB to targets
- Verify target instances are healthy and responding

MAY:
- Check ELB access logs for error patterns
- Review cross-zone load balancing settings

## Phase 2 — Remediate

MUST:
- Add Local Zone subnet to ELB if missing: `aws elbv2 set-subnets --load-balancer-arn <arn> --subnets <subnet-ids>`
- Register Local Zone instances as targets
- Ensure security groups allow ELB health check traffic

SHOULD:
- Configure cross-zone load balancing appropriately
- Set up health checks matching application endpoints

MAY:
- Configure ELB access logging for debugging
- Set up CloudWatch alarms for unhealthy target counts

## Safety Ratings

```
safety_ratings:
  - "describe-load-balancers: GREEN — read-only ELB details"
  - "describe-target-health: GREEN — read-only health check"
  - "set-subnets: YELLOW — changes ELB network configuration"
  - "register-targets: YELLOW — adds instances to load balancer"
  - "delete-load-balancer: RED — removes load balancer, breaks traffic routing"
```

## Escalation Conditions

- ELB type not supported in Local Zone requiring architecture change
- Cross-zone load balancing causing unexpected latency between zones
- ELB health checks failing for all Local Zone targets
- ELB subnet addition requires VPC and security group changes

## Data Sensitivity

- ELB configuration: MEDIUM — reveals architecture and endpoints
- Target health status: LOW — instance health information
- ELB access logs: HIGH — contain client IP addresses and request details
- Security group rules: MEDIUM — reveal allowed traffic patterns

## Prohibited Actions

- NEVER remove subnets from a production ELB without traffic drain
- NEVER disable health checks on target groups
- NEVER expose ELB internal endpoints publicly without security review
- NEVER modify ELB configuration during peak traffic without a rollback plan

## Phase 3 — Rollback

1. If subnet was added incorrectly, remove it: `aws elbv2 set-subnets` with original subnet list
2. If targets were registered incorrectly, deregister them: `aws elbv2 deregister-targets --target-group-arn <arn> --targets Id=<id>`
3. If health check was modified, restore previous health check configuration
4. Monitor ELB metrics for 10 minutes after rollback

## Common Issues

- symptoms: "Cannot add Local Zone subnet to ALB"
  diagnosis: "ALB may not be supported in this Local Zone or subnet is in wrong VPC."
  resolution: "Verify ALB support in the Local Zone. Ensure subnet is in the same VPC as the ALB."

- symptoms: "Targets in Local Zone show unhealthy"
  diagnosis: "Security group blocking health check traffic or application not responding."
  resolution: "Allow ELB security group in target security group. Verify application is running on health check port."

## Output Format

```yaml
root_cause: "elb_integration — <specific_cause>"
evidence:
  - type: elb_config
    content: "<ELB configuration and target health>"
severity: HIGH
mitigation:
  immediate: "Fix ELB subnet or target configuration"
  long_term: "Automate ELB configuration with IaC"
```

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

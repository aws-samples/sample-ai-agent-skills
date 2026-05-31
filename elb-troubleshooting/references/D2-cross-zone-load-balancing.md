---
title: "D2 — Cross-Zone Load Balancing"
description: "Diagnose and resolve ELB cross-zone load balancing configuration and distribution issues"
status: active
severity: MEDIUM
triggers:
  - "cross-zone load balancing"
  - "uneven distribution"
  - "AZ imbalance"
  - "traffic distribution"
owner: devops-agent
objective: "Resolve cross-zone load balancing issues and ensure even traffic distribution across targets"
context: "Cross-zone load balancing distributes traffic evenly across all registered targets in all enabled AZs, regardless of the AZ of the load balancer node receiving the request. ALB has cross-zone enabled by default (can be disabled per target group). NLB has cross-zone disabled by default. Without cross-zone, each LB node distributes traffic only to targets in its own AZ, which can cause uneven distribution if AZs have different numbers of targets."
---

## Phase 1 — Triage

MUST:
- Check cross-zone setting on ALB: `aws elbv2 describe-load-balancer-attributes --load-balancer-arn <lb-arn> --query 'Attributes[?Key==\`load_balancing.cross_zone.enabled\`]'`
- Check cross-zone setting on target group (ALB): `aws elbv2 describe-target-group-attributes --target-group-arn <tg-arn> --query 'Attributes[?Key==\`load_balancing.cross_zone.enabled\`]'`
- Verify target distribution across AZs: `aws elbv2 describe-target-health --target-group-arn <tg-arn>` and check AZ of each target
- Check load balancer AZ configuration: `aws elbv2 describe-load-balancers --load-balancer-arns <lb-arn> --query 'LoadBalancers[0].AvailabilityZones'`
- Review CloudWatch RequestCount per AZ if available

SHOULD:
- Compare the number of healthy targets per AZ
- Check if Auto Scaling group is balanced across AZs: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg> --query 'AutoScalingGroups[0].Instances[*].AvailabilityZone'`
- Verify DNS is distributing traffic across LB nodes in different AZs

MAY:
- Check if zonal shift or zonal autoshift is active: `aws arc-zonal-shift list-zonal-shifts`
- Review NLB per-AZ metrics for traffic distribution
- Check if client-side DNS caching is causing AZ affinity

## Phase 2 — Remediate

MUST:
- If cross-zone disabled and uneven distribution: enable cross-zone on ALB: `aws elbv2 modify-load-balancer-attributes --load-balancer-arn <lb-arn> --attributes Key=load_balancing.cross_zone.enabled,Value=true`
- For NLB: enable cross-zone: `aws elbv2 modify-target-group-attributes --target-group-arn <tg-arn> --attributes Key=load_balancing.cross_zone.enabled,Value=true`
- If AZ has no targets: add targets in that AZ or remove the AZ from the LB
- If target count is unbalanced: adjust Auto Scaling to maintain equal capacity per AZ

SHOULD:
- Verify traffic distribution improves after enabling cross-zone
- Ensure Auto Scaling group has AZ rebalancing enabled
- Monitor per-target request counts after changes

MAY:
- Consider disabling cross-zone for NLB if AZ isolation is required for fault tolerance
- Implement weighted target groups for intentional uneven distribution
- Document the cross-zone configuration rationale

## Common Issues

- symptoms: "Some targets receive much more traffic than others"
  diagnosis: "Cross-zone disabled with unequal target counts per AZ."
  resolution: "Enable cross-zone load balancing or equalize target counts across AZs."

- symptoms: "NLB targets in one AZ are overloaded"
  diagnosis: "NLB cross-zone is disabled by default — each AZ node only routes to local targets."
  resolution: "Enable cross-zone on the NLB target group, or ensure equal targets per AZ."

- symptoms: "Traffic drops to zero for targets in one AZ"
  diagnosis: "LB node in that AZ is not receiving DNS traffic, or the AZ was removed from the LB."
  resolution: "Verify the LB has subnets in all required AZs. Check DNS resolution."

- symptoms: "Enabling cross-zone increases latency"
  diagnosis: "Cross-AZ traffic adds network latency and incurs data transfer costs."
  resolution: "This is expected. If latency-sensitive, keep cross-zone disabled and balance targets per AZ."

## Output Format

```yaml
root_cause: "Cross-zone load balancing issue — <specific_cause>"
evidence:
  - type: cross_zone_config
    content: "<cross-zone attribute settings>"
  - type: target_distribution
    content: "<target count per AZ>"
severity: MEDIUM
mitigation:
  immediate: "Enable/disable cross-zone or balance targets across AZs"
  long_term: "Implement AZ-balanced Auto Scaling and monitoring"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get): GREEN — read-only"
  - "Phase 2 cross-zone toggle: YELLOW — changes traffic distribution pattern"
  - "Phase 2 AZ changes on LB: YELLOW — affects traffic routing"

## Escalation Conditions

escalation_conditions:
  - "Cross-zone change significantly alters traffic patterns in production"
  - "AZ removal from LB affects service availability"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Cross-zone data transfer costs are a concern"
  - "Issue involves NLB zonal isolation requirements for compliance"

## Data Sensitivity

data_sensitivity:
  - command: "describe-load-balancer-attributes"
    sensitivity: LOW
    contains: "Load balancer configuration settings"
  - command: "describe-target-health"
    sensitivity: LOW
    contains: "Target IDs and AZ distribution"
  - command: "CloudWatch metrics"
    sensitivity: LOW
    contains: "Aggregated traffic distribution metrics"

## Prohibited Actions

prohibited_actions:
  - "NEVER remove an AZ from a production LB without verifying targets exist in remaining AZs"
  - "NEVER enable cross-zone on NLB without understanding the data transfer cost implications"
  - "NEVER disable cross-zone on ALB during peak traffic without a rollback plan"
  - "NEVER assume cross-zone is enabled — verify the actual configuration"

## Phase 3 — Rollback

- Before changing cross-zone settings, document the current configuration
- If enabling cross-zone causes latency issues, disable it and balance targets per AZ instead
- If disabling cross-zone causes overload in some AZs, re-enable it
- If AZ changes were made to the LB, restore the previous AZ configuration
- Monitor traffic distribution and latency for 15 minutes after any changes

---
title: "B2 — NLB Requirements"
description: "Diagnose NLB configuration issues for VPC endpoint services"
status: active
severity: HIGH
triggers:
  - "NLB for PrivateLink"
  - "load balancer type"
  - "NLB health check"
  - "NLB targets unhealthy"
  - "NLB not working with endpoint service"
owner: devops-agent
objective: "Resolve NLB configuration issues that affect endpoint service functionality"
context: "Endpoint services require an NLB with healthy targets. The NLB must have listeners configured and targets registered. Cross-zone load balancing affects how traffic is distributed. NLB health checks determine target availability. Unhealthy targets cause endpoint service connectivity failures."
---

## Phase 1 — Triage

MUST:
- Verify NLB type and state: `aws elbv2 describe-load-balancers --load-balancer-arns <nlb-arn>`
- Check NLB listeners: `aws elbv2 describe-listeners --load-balancer-arn <nlb-arn>`
- Check target group health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- Verify NLB is associated with the endpoint service: `aws ec2 describe-vpc-endpoint-service-configurations --service-ids <service-id>`

SHOULD:
- Check target group configuration: `aws elbv2 describe-target-groups --target-group-arns <tg-arn>`
- Verify NLB subnets cover the required AZs
- Check cross-zone load balancing setting: `aws elbv2 describe-load-balancer-attributes --load-balancer-arn <nlb-arn>`

MAY:
- Review NLB access logs for traffic patterns
- Check NLB CloudWatch metrics for connection counts

## Phase 2 — Remediate

MUST:
- Ensure NLB has at least one listener on the required port
- Register healthy targets in the target group
- Fix health check configuration if targets are unhealthy

SHOULD:
- Enable cross-zone load balancing for even distribution
- Place NLB in multiple AZs for high availability
- Configure appropriate health check intervals and thresholds

MAY:
- Enable NLB access logs for troubleshooting
- Set up CloudWatch alarms for unhealthy target count

## Common Issues

- symptoms: "All targets unhealthy in NLB target group"
  diagnosis: "Health check failing. Targets not responding on health check port/path."
  resolution: "Fix health check configuration. Verify targets are running and accessible on the health check port."

- symptoms: "Endpoint service traffic only reaching some AZs"
  diagnosis: "Cross-zone load balancing disabled. NLB only routes to targets in the same AZ."
  resolution: "Enable cross-zone load balancing or ensure targets in all AZs."

- symptoms: "Connection timeout through endpoint service"
  diagnosis: "NLB listener not configured or targets not registered."
  resolution: "Add listener on the correct port. Register targets in the target group."

## Output Format

```yaml
root_cause: "nlb_requirements — <specific_cause>"
evidence:
  - type: nlb_config
    content: "<NLB type, listeners, subnets>"
  - type: target_health
    content: "<target group health status>"
  - type: cross_zone
    content: "<cross-zone load balancing setting>"
severity: HIGH
mitigation:
  immediate: "Fix NLB configuration and target health"
  long_term: "Implement NLB monitoring and health check optimization"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — NLB configuration is infrastructure-level
- Service Disruption Risk: MEDIUM — NLB issues affect endpoint service availability
- Compliance Impact: MEDIUM — NLB configuration affects service security

## Escalation Conditions
- NLB health checks fail for all targets
- NLB configuration incompatible with endpoint service
- NLB cross-zone load balancing issues
- NLB TLS configuration errors
- NLB capacity insufficient for traffic

## Data Sensitivity
- NLB configurations — INTERNAL
- Target group details — INTERNAL
- Health check configurations — INTERNAL
- TLS certificate details — INTERNAL
- Never expose NLB internal configurations externally

## Prohibited Actions
- Do NOT disable NLB health checks
- Do NOT modify NLB configurations without testing
- Do NOT use NLB without TLS for sensitive traffic
- Do NOT ignore NLB health check failures
- Do NOT bypass NLB security configurations
- Do NOT delete NLBs attached to active endpoint services

## Phase 3 — Rollback
- Restore previous NLB configurations
- Revert target group changes
- Restore health check configurations
- Verify NLB health after changes
- Document NLB issues and configurations
- Update NLB management procedures

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

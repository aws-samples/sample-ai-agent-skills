---
title: "E1 — Managed Scaling Issues"
description: "Diagnose EMR managed scaling problems"
status: active
severity: MEDIUM
triggers:
  - "scaling not working"
  - "cluster not scaling"
  - "managed scaling error"
  - "scaling too slow"
owner: devops-agent
objective: "Identify and resolve EMR managed scaling issues"
context: "Managed scaling automatically adjusts cluster capacity based on YARN metrics. It has min/max unit limits, cooldown periods, and uses YARN pending containers and utilization for decisions. Scaling is not instant — provisioning takes minutes."
---

## Phase 1 — Triage

MUST:
- Check managed scaling policy: `aws emr get-managed-scaling-policy --cluster-id <id>`
- Verify current instance group sizes: `aws emr list-instance-groups --cluster-id <id>`
- Check CloudWatch scaling metrics: YARNMemoryAvailablePercentage, ContainerPendingRatio
- Verify scaling is enabled on the cluster

SHOULD:
- Check for scaling events in CloudTrail
- Verify min/max unit configuration
- Check for EC2 capacity issues in the AZ
- Review scaling cooldown timing

MAY:
- Check CloudWatch metrics for scaling decision patterns
- Review instance type availability

## Phase 2 — Remediate

MUST:
- Configure appropriate min/max units for the workload
- Ensure managed scaling policy is attached to the cluster
- Verify EC2 instance types are available in the AZ

SHOULD:
- Use instance fleets with multiple instance types for better scaling
- Set appropriate min units for baseline capacity
- Monitor scaling behavior and adjust limits

MAY:
- Implement custom scaling based on application metrics
- Create scaling configuration templates

## Common Issues

- symptoms: "Cluster not scaling up despite pending containers"
  diagnosis: "Max units reached or EC2 capacity unavailable."
  resolution: "Increase max units. Use instance fleets with multiple types."

- symptoms: "Scaling too slow for bursty workloads"
  diagnosis: "Instance provisioning takes minutes, cooldown adds delay."
  resolution: "Pre-provision with higher min units. Use pre-initialized capacity."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Configure min/max units for workload | YELLOW | Changes scaling boundaries — affects capacity and cost |
| Attach managed scaling policy | GREEN | Enables auto-scaling — non-destructive |
| Use instance fleets with multiple types | GREEN | Improves scaling flexibility — non-destructive |
| Implement custom scaling based on metrics | YELLOW | Custom scaling logic — test thoroughly |
| Pre-provision with higher min units | YELLOW | Increases baseline cost |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Scaling failure causes job delays affecting SLA
- Scaling changes significantly impact cost

## Data Sensitivity

- **Classification: HIGH**
- Scaling metrics reveal workload patterns and data processing volumes
- YARN metrics expose resource utilization and application behavior
- Instance fleet configuration reveals cost optimization strategy
- CloudWatch scaling metrics reveal peak usage patterns

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest setting max units to unlimited without cost controls
- **NEVER** suggest disabling managed scaling on production clusters without a replacement strategy

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Changed min/max scaling units | Revert to previous min/max values |
| Attached managed scaling policy | Remove managed scaling policy if not needed |
| Increased min units for warm capacity | Reduce min units to previous value to lower cost |

## Output Format

```yaml
root_cause: "managed_scaling — <specific_cause>"
evidence:
  - type: scaling_policy
    content: "<managed scaling configuration>"
  - type: scaling_metrics
    content: "<YARN metrics and scaling decisions>"
severity: MEDIUM
mitigation:
  immediate: "Adjust scaling configuration"
  long_term: "Optimize scaling parameters based on workload patterns"
```

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
  - "NEVER suggest disabling encryption"
  - "NEVER suggest public subnet placement for clusters"
  - "NEVER suggest removing Kerberos authentication"

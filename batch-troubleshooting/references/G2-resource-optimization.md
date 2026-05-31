---
title: "G2 — Resource Optimization"
description: "Optimize resource allocation and cost efficiency"
status: active
severity: LOW
triggers:
  - "resource waste"
  - "over-provisioned"
  - "cost optimization"
  - "right-sizing"
  - "resource utilization"
owner: devops-agent
objective: "Optimize resource allocation for cost and performance"
context: "Resource optimization includes: right-sizing vCPU/memory, using Spot instances, optimizing instance types, reducing idle capacity, and improving job packing."
---

## Phase 1 — Triage

MUST:
- Check job resource utilization vs allocation
- Check CE instance utilization: `aws cloudwatch get-metric-statistics --namespace AWS/Batch --metric-name CPUUtilization --dimensions Name=ComputeEnvironment,Value=<ce-name> --start-time <start> --end-time <end> --period 3600 --statistics Average`
- Check minvCpus setting (idle cost)
- Review Spot vs On-Demand usage

SHOULD:
- Analyze job memory usage vs allocation (check for OOM vs waste)
- Check instance type selection efficiency
- Review job duration patterns

MAY:
- Calculate cost per job
- Compare Spot savings vs interruption impact

## Phase 2 — Remediate

MUST:
- Right-size job vCPU and memory based on actual usage
- Set minvCpus to 0 if warm capacity not needed
- Use Spot instances for fault-tolerant workloads

SHOULD:
- Use BEST_FIT_PROGRESSIVE allocation for efficient packing
- Implement Spot with retry strategy
- Use Fargate for variable workloads (pay per job)

MAY:
- Implement cost allocation tags
- Set up cost monitoring dashboards

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Right-size job vCPU and memory | GREEN | Job definition tuning — non-destructive |
| Set minvCpus to 0 | GREEN | Cost optimization — non-destructive |
| Use Spot instances for fault-tolerant workloads | YELLOW | Spot may be interrupted — ensure retry strategy |
| Use BEST_FIT_PROGRESSIVE allocation | GREEN | Optimization — non-destructive |
| Implement cost allocation tags | GREEN | Tagging — non-destructive |

## Escalation Conditions

- Compute environment runs production batch jobs
- Resource optimization changes affect job reliability
- Cost reduction targets require significant architecture changes
- Spot migration requires retry strategy implementation

## Data Sensitivity

- **Classification: MEDIUM**
- Resource utilization metrics reveal workload characteristics
- Cost analysis exposes infrastructure spending patterns
- Instance type selection reveals compute optimization strategy
- Spot vs On-Demand usage reveals cost sensitivity

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest reducing job resources below what causes OOM or failures
- **NEVER** suggest using only Spot for critical production jobs without fallback
- **NEVER** suggest removing monitoring to reduce costs

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Right-sized job resources | Increase resources if jobs start failing with OOM |
| Set minvCpus to 0 | Increase minvCpus if warm capacity is needed |
| Switched to Spot instances | Revert to On-Demand if Spot interruptions are unacceptable |

## Output Format

```yaml
root_cause: "resource_optimization — <specific_area>"
evidence:
  - type: utilization
    content: "<resource utilization metrics>"
  - type: cost_analysis
    content: "<cost breakdown>"
severity: LOW
mitigation:
  immediate: "Right-size resources and use Spot"
  long_term: "Implement continuous optimization and cost monitoring"
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
  - "NEVER suggest overly broad compute environment permissions"
  - "NEVER suggest disabling job queue scheduling"
  - "NEVER suggest public subnet placement for compute environments"

---
title: "F2 — Capacity Issues"
description: "Diagnose EC2 capacity problems for ParallelCluster compute fleet"
status: active
severity: HIGH
triggers:
  - "InsufficientInstanceCapacity"
  - "capacity"
  - "no instances available"
  - "spot capacity"
owner: devops-agent
objective: "Resolve EC2 capacity constraints affecting ParallelCluster scaling"
context: "ParallelCluster compute nodes are EC2 instances. Capacity depends on instance type availability in the target AZ. Spot instances may be interrupted. Multi-instance-type and multi-AZ configurations improve capacity resilience."
---

## Phase 1 — Triage

MUST:
- Check slurm_resume.log for capacity errors: `pcluster get-cluster-log-events --cluster-name <cluster-name> --log-stream-name ip-<ip>.i-<id>.slurm_resume`
- Check EC2 service quotas: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A`
- Verify instance type availability: `aws ec2 describe-instance-type-offerings --location-type availability-zone --filters "Name=location,Values=<az>" "Name=instance-type,Values=<type>"`
- Check spot pricing if using spot: `aws ec2 describe-spot-price-history --instance-types <type> --availability-zone <az> --max-items 5`

SHOULD:
- Check for spot interruption notices
- Review cluster config for instance type flexibility
- Check multiple AZs for capacity

MAY:
- Check AWS Health Dashboard for capacity events
- Review historical capacity patterns

## Phase 2 — Remediate

MUST:
- If on-demand capacity: try different instance types or AZs
- If spot interrupted: configure on-demand fallback or multiple spot pools
- If quota limit: request quota increase

SHOULD:
- Configure multiple instance types per queue (Allocation Strategy)
- Use capacity-optimized allocation strategy for spot
- Spread across multiple AZs

MAY:
- Use capacity reservations for critical workloads
- Implement job preemption for priority management

## Common Issues

- symptoms: "InsufficientInstanceCapacity in slurm_resume.log"
  diagnosis: "No available capacity for the requested instance type in the AZ."
  resolution: "Add alternative instance types to the queue. Use multiple AZs."

- symptoms: "Spot instances terminated mid-job"
  diagnosis: "Spot interruption — capacity reclaimed by AWS."
  resolution: "Use on-demand for critical jobs. Implement checkpointing for spot jobs."

## Output Format

```yaml
root_cause: "capacity — <specific_cause>"
evidence:
  - type: launch_errors
    content: "<slurm_resume error messages>"
  - type: quotas
    content: "<EC2 service quotas>"
severity: HIGH
mitigation:
  immediate: "Use alternative instance types or AZs"
  long_term: "Configure multi-instance-type queues and capacity reservations"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — capacity management is operational
- Service Disruption Risk: MEDIUM — capacity issues affect job throughput
- Compliance Impact: LOW — capacity is an operational concern

## Escalation Conditions
- Insufficient capacity for critical job execution
- Capacity reservations not being honored
- Spot instance interruptions affecting job completion
- On-Demand capacity unavailable in target AZ
- Capacity costs exceed budget thresholds

## Data Sensitivity
- Capacity utilization metrics — INTERNAL
- Cost allocation data — INTERNAL
- Instance type availability — INTERNAL
- Capacity reservation details — INTERNAL
- Never expose capacity details that reveal research scale

## Prohibited Actions
- Do NOT rely solely on Spot instances for critical jobs
- Do NOT ignore capacity warnings
- Do NOT exceed budget thresholds without approval
- Do NOT reserve capacity without cost analysis
- Do NOT bypass capacity planning procedures
- Do NOT ignore Spot interruption patterns

## Phase 3 — Rollback
- Restore previous capacity configurations
- Revert capacity reservation changes
- Switch from Spot to On-Demand for critical jobs if needed
- Restore budget alert configurations
- Verify capacity availability after changes
- Document capacity issues and planning updates

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

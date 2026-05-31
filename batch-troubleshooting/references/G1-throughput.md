---
title: "G1 — Throughput Issues"
description: "Diagnose job throughput bottlenecks"
status: active
severity: MEDIUM
triggers:
  - "low throughput"
  - "jobs slow"
  - "processing bottleneck"
  - "queue backlog"
  - "scaling too slow"
owner: devops-agent
objective: "Identify and resolve job throughput bottlenecks"
context: "Throughput issues include: CE scaling too slow, maxvCpus too low, instance types too small, job submission rate limits, or queue scheduling delays."
---

## Phase 1 — Triage

MUST:
- Check RUNNABLE job count: `aws batch list-jobs --job-queue <queue-name> --job-status RUNNABLE --query 'jobSummaryList | length(@)'`
- Check CE capacity: `aws batch describe-compute-environments --compute-environments <ce-name> --query 'computeEnvironments[*].computeResources.{Min:minvCpus,Max:maxvCpus,Desired:desiredvCpus}'`
- Check job completion rate over time
- Verify CE is scaling to meet demand

SHOULD:
- Check for API throttling on job submission
- Verify instance launch success rate
- Check for resource contention

MAY:
- Analyze job duration distribution
- Check for I/O bottlenecks in jobs

## Phase 2 — Remediate

MUST:
- Increase maxvCpus if CE is at capacity limit
- Add more instance types for faster scaling
- Use optimal allocation strategy (BEST_FIT_PROGRESSIVE recommended)

SHOULD:
- Set minvCpus > 0 for warm capacity
- Use multiple CEs for different workload types
- Optimize job resource requests to maximize packing

MAY:
- Implement job batching for small jobs
- Use array jobs for embarrassingly parallel workloads

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Increase maxvCpus for CE capacity | YELLOW | Increases capacity ceiling — cost increase |
| Add more instance types for faster scaling | GREEN | CE configuration improvement — non-destructive |
| Use BEST_FIT_PROGRESSIVE allocation | GREEN | Optimization — non-destructive |
| Set minvCpus > 0 for warm capacity | YELLOW | Maintains idle instances — ongoing cost |
| Optimize job resource requests | GREEN | Job definition tuning — non-destructive |

## Escalation Conditions

- Compute environment runs production batch jobs
- Throughput bottleneck causes growing job backlog affecting SLA
- CE scaling too slow for bursty workloads
- API throttling on job submission

## Data Sensitivity

- **Classification: MEDIUM**
- Queue backlog metrics reveal workload volume and patterns
- CE capacity utilization exposes infrastructure sizing
- Job completion rates reveal processing throughput
- Instance launch patterns reveal scaling behavior

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest setting maxvCpus to unlimited without cost controls
- **NEVER** suggest reducing job resource requests below actual needs to improve packing
- **NEVER** suggest disabling CE scaling to resolve throughput issues

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Increased maxvCpus | Reduce maxvCpus to previous value after backlog clears |
| Set minvCpus > 0 | Set minvCpus back to 0 to stop incurring idle costs |
| Changed allocation strategy | Revert to previous allocation strategy |

## Output Format

```yaml
root_cause: "throughput_issue — <specific_cause>"
evidence:
  - type: queue_backlog
    content: "<RUNNABLE job count>"
  - type: ce_capacity
    content: "<CE capacity utilization>"
severity: MEDIUM
mitigation:
  immediate: "Increase CE capacity or optimize resource requests"
  long_term: "Implement capacity planning and monitoring"
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

---
title: "A2 — Jobs Stuck in RUNNABLE"
description: "Diagnose why jobs are stuck in RUNNABLE state"
status: active
severity: HIGH
triggers:
  - "job stuck RUNNABLE"
  - "job not starting"
  - "waiting for resources"
  - "RUNNABLE for hours"
  - "job queued"
owner: devops-agent
objective: "Identify why jobs are stuck in RUNNABLE and resolve the issue"
context: "RUNNABLE means the job is waiting for compute resources. Causes include: CE not scaling, instance types unavailable, resource requests too large, VPC/subnet issues, service quotas reached, or CE in INVALID state."
---

## Phase 1 — Triage

MUST:
- Check job status: `aws batch describe-jobs --jobs <job-id> --query 'jobs[*].{Status:status,Reason:statusReason,CreatedAt:createdAt}'`
- Check compute environment state: `aws batch describe-compute-environments --compute-environments <ce-name> --query 'computeEnvironments[*].{State:state,Status:status,StatusReason:statusReason,DesiredvCpus:computeResources.desiredvCpus,MaxvCpus:computeResources.maxvCpus}'`
- Check job resource requirements vs CE capabilities
- Check job queue is ENABLED and associated with a valid CE: `aws batch describe-job-queues --job-queues <queue-name>`

SHOULD:
- Check if CE is at maxvCpus limit
- Verify instance types are available in the configured subnets/AZs
- Check EC2 service quotas: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A`
- Verify VPC subnets have available IPs

MAY:
- Check for CE scaling events in CloudTrail
- Verify ECS cluster has registered instances (for EC2 CE)

## Phase 2 — Remediate

MUST:
- If CE INVALID: fix the CE configuration issue (VPC, subnets, IAM)
- If at maxvCpus: increase maxvCpus or reduce running jobs
- If resource request too large: reduce job vCPU/memory or add larger instance types
- If instance unavailable: add more instance types or AZs

SHOULD:
- Update CE: `aws batch update-compute-environment --compute-environment <ce-name> --compute-resources maxvCpus=256`
- Add instance type diversity for better availability
- Use optimal instance type selection

MAY:
- Set minvCpus > 0 for warm capacity
- Implement job queue monitoring for RUNNABLE duration

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix CE configuration (VPC, subnets, IAM) | YELLOW | CE update — may cause brief scaling pause |
| Increase maxvCpus | YELLOW | Increases capacity ceiling — cost increase |
| Reduce job vCPU/memory requirements | GREEN | Job definition change — non-destructive |
| Add more instance types or AZs | GREEN | CE configuration improvement — non-destructive |
| Set minvCpus > 0 for warm capacity | YELLOW | Maintains idle instances — ongoing cost |

## Escalation Conditions

- Compute environment runs production batch jobs
- Jobs stuck in RUNNABLE for extended period affecting SLA
- CE in INVALID state blocking all job execution
- EC2 service quota limits reached

## Data Sensitivity

- **Classification: MEDIUM**
- CE configuration reveals infrastructure architecture and instance types
- Job resource requirements expose workload characteristics
- Service quota usage reveals organizational scale
- VPC/subnet details expose network architecture

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest setting maxvCpus to unlimited without cost controls
- **NEVER** suggest reducing resource requirements below what jobs actually need
- **NEVER** suggest disabling CE to resolve INVALID state while jobs are queued

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Updated CE configuration | Revert CE configuration to previous settings |
| Increased maxvCpus | Reduce maxvCpus to previous value after issue resolves |
| Set minvCpus > 0 | Set minvCpus back to 0 to stop incurring idle costs |

## Output Format

```yaml
root_cause: "stuck_runnable — <specific_cause>"
evidence:
  - type: ce_status
    content: "<CE state and capacity>"
  - type: resource_request
    content: "<job resource requirements>"
severity: HIGH
mitigation:
  immediate: "Fix CE scaling or resource request mismatch"
  long_term: "Implement CE monitoring and capacity planning"
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

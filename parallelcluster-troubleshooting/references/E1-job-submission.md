---
title: "E1 — Job Submission Issues"
description: "Diagnose job submission failures in ParallelCluster"
status: active
severity: HIGH
triggers:
  - "sbatch failed"
  - "job submission"
  - "cannot submit job"
  - "job pending"
owner: devops-agent
objective: "Resolve job submission errors and pending job issues"
context: "Jobs are submitted via Slurm commands (sbatch, srun, salloc). Submission failures may be due to partition configuration, resource requests, user permissions, or Slurm controller issues."
---

## Phase 1 — Triage

MUST:
- Check job queue: `squeue -l`
- Check pending job reasons: `squeue -t PENDING -o "%.18i %.9P %.8j %.8u %.2t %.10M %.6D %R"`
- Verify partition is available: `sinfo`
- Check job submission script for errors
- Verify user has access to the partition: `sacctmgr show associations user=<user>`

SHOULD:
- Check resource availability: `sinfo -N -l`
- Verify requested resources don't exceed partition limits
- Check Slurm controller logs: `cat /var/log/slurmctld.log | tail -20`

MAY:
- Review Slurm QOS limits if configured
- Check for reservation conflicts

## Phase 2 — Remediate

MUST:
- If partition down: check slurmctld and restart if needed
- If resources unavailable: adjust job resource requests or wait for nodes
- If permission denied: add user to Slurm account/partition

SHOULD:
- Validate job scripts before submission
- Use appropriate partition for the workload
- Set reasonable resource requests (CPUs, memory, time)

MAY:
- Implement job templates for common workloads
- Set up Slurm accounting for resource tracking

## Common Issues

- symptoms: "sbatch: error: Batch job submission failed: Invalid partition"
  diagnosis: "Partition name in sbatch does not match configured partitions."
  resolution: "Check available partitions with sinfo. Use correct partition name."

- symptoms: "Job stuck in PD with (Resources) reason"
  diagnosis: "Requested resources exceed available capacity. Nodes are launching."
  resolution: "Wait for dynamic nodes to launch (2-5 min). Check slurm_resume.log."

## Output Format

```yaml
root_cause: "job_submission — <specific_cause>"
evidence:
  - type: squeue
    content: "<job queue status>"
  - type: sinfo
    content: "<partition and node status>"
severity: HIGH
mitigation:
  immediate: "Fix submission error or resource request"
  long_term: "Implement job templates and submission validation"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: MEDIUM — job scripts may contain sensitive parameters
- Service Disruption Risk: LOW — submission failures affect individual jobs
- Compliance Impact: LOW — job submission is an operational concern

## Escalation Conditions
- All job submissions fail simultaneously
- Job submission queue becomes unresponsive
- Permission errors prevent authorized users from submitting jobs
- Job submission parameters cause scheduler instability
- Submission rate exceeds scheduler capacity

## Data Sensitivity
- Job scripts and parameters — INTERNAL (may contain CONFIDENTIAL data)
- User credentials — RESTRICTED
- Job resource requests — INTERNAL
- Queue configurations — INTERNAL
- Never expose job scripts containing credentials or sensitive data

## Prohibited Actions
- Do NOT submit jobs with hardcoded credentials
- Do NOT bypass job resource limits
- Do NOT submit untested jobs to production queues
- Do NOT ignore job submission errors
- Do NOT grant job submission access to unauthorized users
- Do NOT submit jobs that exceed cluster capacity without approval

## Phase 3 — Rollback
- Cancel incorrectly submitted jobs
- Restore queue configurations if modified
- Revert permission changes that affected job submission
- Re-enable job submission if accidentally disabled
- Verify job submission pipeline after changes
- Document submission issues and resolutions

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

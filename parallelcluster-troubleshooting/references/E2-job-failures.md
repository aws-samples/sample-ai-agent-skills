---
title: "E2 — Job Execution Failures"
description: "Diagnose job runtime failures and errors"
status: active
severity: HIGH
triggers:
  - "job failed"
  - "job error"
  - "FAILED state"
  - "job timeout"
  - "OOM"
owner: devops-agent
objective: "Identify and resolve job execution failures"
context: "Job failures may be due to application errors, resource limits (OOM, timeout), storage issues, or node problems. Slurm job accounting and output files contain diagnostic information."
---

## Phase 1 — Triage

MUST:
- Check job status: `sacct -j <job-id> --format=JobID,JobName,State,ExitCode,MaxRSS,Elapsed,NodeList`
- Check job output file: `cat slurm-<job-id>.out`
- Check job error file: `cat slurm-<job-id>.err`
- Check node status where job ran: `scontrol show node <node-name>`

SHOULD:
- Check for OOM killer: `sacct -j <job-id> --format=JobID,State,MaxRSS,ReqMem`
- Check job time limit: `sacct -j <job-id> --format=JobID,Timelimit,Elapsed`
- Verify shared storage was accessible during job execution

MAY:
- Check dmesg on the compute node for kernel-level errors
- Review application-specific logs

## Phase 2 — Remediate

MUST:
- If OOM: increase memory request or optimize application memory usage
- If timeout: increase time limit or optimize application performance
- If node failure: resubmit job — Slurm will use a different node

SHOULD:
- Set appropriate resource limits in job scripts
- Use --mem-per-cpu for memory-proportional allocation
- Implement job checkpointing for long-running jobs

MAY:
- Use Slurm job arrays for parameter sweeps
- Implement automatic job resubmission on failure

## Common Issues

- symptoms: "Job exits with signal 9 (SIGKILL)"
  diagnosis: "OOM killer terminated the process — exceeded memory limit."
  resolution: "Increase --mem or --mem-per-cpu. Check MaxRSS with sacct."

- symptoms: "Job exits with TIMEOUT state"
  diagnosis: "Job exceeded the time limit specified in sbatch."
  resolution: "Increase --time limit or optimize application performance."

## Output Format

```yaml
root_cause: "job_failure — <specific_cause>"
evidence:
  - type: sacct
    content: "<job accounting data>"
  - type: job_output
    content: "<relevant output/error>"
severity: HIGH
mitigation:
  immediate: "Fix resource limits or application error and resubmit"
  long_term: "Implement checkpointing and resource estimation"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — job output may contain sensitive results
- Service Disruption Risk: MEDIUM — job failures affect research timelines
- Compliance Impact: LOW — job failures are operational concerns

## Escalation Conditions
- Jobs fail with OOM errors repeatedly
- Job failures correlate with infrastructure issues
- Critical research jobs fail affecting deadlines
- Job failure rate exceeds normal baseline
- Jobs fail with permission errors after cluster changes

## Data Sensitivity
- Job output and results — CONFIDENTIAL
- Error logs — INTERNAL (may contain CONFIDENTIAL data)
- Job resource utilization — INTERNAL
- Core dump files — CONFIDENTIAL
- Never expose research results or job outputs

## Prohibited Actions
- Do NOT retry failed jobs without understanding the root cause
- Do NOT increase job resource limits without capacity analysis
- Do NOT ignore persistent job failure patterns
- Do NOT clear job accounting data
- Do NOT expose job error logs containing sensitive data
- Do NOT modify job parameters in production without testing

## Phase 3 — Rollback
- Re-submit failed jobs with corrected parameters
- Restore previous job configurations
- Revert cluster changes that caused job failures
- Restore job output data from checkpoints if available
- Verify job execution environment after changes
- Document job failure patterns and resolutions

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

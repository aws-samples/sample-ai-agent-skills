---
title: "B2 — Slurm Scheduler Issues"
description: "Diagnose Slurm scheduler configuration and operation problems"
status: active
severity: HIGH
triggers:
  - "Slurm"
  - "slurmctld"
  - "scheduler error"
  - "partition"
  - "sinfo"
owner: devops-agent
objective: "Resolve Slurm scheduler configuration, partition, and operation issues"
context: "ParallelCluster 3.x uses Slurm exclusively. The Slurm controller (slurmctld) runs on the head node. Partitions map to ParallelCluster queues. Slurm manages node power states for dynamic scaling."
---

## Phase 1 — Triage

MUST:
- Check Slurm status (on head node): `sinfo`
- Check partition details: `scontrol show partitions`
- Check node details: `scontrol show nodes`
- Check slurmctld log: `cat /var/log/slurmctld.log | tail -50`
- Check job queue: `squeue -l`

SHOULD:
- Verify Slurm configuration: `scontrol show config`
- Check for down or drained nodes: `sinfo -R`
- Review Slurm accounting: `sacct -a --starttime=<date>`

MAY:
- Check slurmdbd status if using external accounting
- Review Slurm priority configuration: `scontrol show config | grep Priority`

## Phase 2 — Remediate

MUST:
- If slurmctld down: restart it: `sudo systemctl restart slurmctld`
- If nodes drained: investigate reason and undrain: `sudo scontrol update nodename=<node> state=idle`
- If partition down: check partition configuration and enable it

SHOULD:
- Verify Slurm configuration matches ParallelCluster config
- Do NOT manually edit slurm.conf — use pcluster update-cluster instead
- Check for Slurm version compatibility issues

MAY:
- Implement Slurm accounting for job tracking
- Configure fair-share scheduling for multi-user environments

## Common Issues

- symptoms: "All partitions show as down"
  diagnosis: "slurmctld service not running or configuration error."
  resolution: "Check slurmctld status and logs. Restart if needed."

- symptoms: "Jobs stuck in PD (pending) state with (Priority) reason"
  diagnosis: "No available nodes or all nodes are down/drained."
  resolution: "Check sinfo for node states. Fix drained nodes or wait for dynamic node launch."

## Output Format

```yaml
root_cause: "slurm — <specific_cause>"
evidence:
  - type: sinfo
    content: "<partition and node status>"
  - type: slurmctld_log
    content: "<relevant log entries>"
severity: HIGH
mitigation:
  immediate: "Fix Slurm configuration or restart services"
  long_term: "Implement Slurm monitoring and accounting"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — scheduler manages job queuing
- Service Disruption Risk: HIGH — scheduler failures halt all job processing
- Compliance Impact: MEDIUM — scheduler configuration affects job isolation and resource allocation

## Escalation Conditions
- Slurm scheduler daemon crashes or becomes unresponsive
- Job scheduling stops completely
- Scheduler database corruption detected
- Scheduler configuration changes cause widespread job failures
- Scheduler accounting data lost

## Data Sensitivity
- Scheduler configuration — INTERNAL
- Job submission parameters — INTERNAL (may contain CONFIDENTIAL data)
- User job data — CONFIDENTIAL
- Scheduler accounting records — INTERNAL
- Never expose scheduler admin credentials

## Prohibited Actions
- Do NOT modify slurm.conf on production clusters without testing
- Do NOT restart slurmctld during active job execution without draining
- Do NOT delete scheduler accounting data
- Do NOT bypass scheduler resource limits
- Do NOT grant scheduler admin access to unauthorized users
- Do NOT modify partition configurations during peak usage

## Phase 3 — Rollback
- Restore previous slurm.conf from backup
- Restart scheduler daemons after configuration rollback
- Restore scheduler accounting database if corrupted
- Re-queue failed jobs after scheduler recovery
- Verify scheduler health and job processing after rollback
- Document scheduler issues and configuration changes

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

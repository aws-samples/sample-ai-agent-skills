---
title: "F1 — Auto-Scaling Issues"
description: "Diagnose auto-scaling problems in ParallelCluster compute fleet"
status: active
severity: HIGH
triggers:
  - "auto-scaling"
  - "scaling"
  - "nodes not launching"
  - "scale down"
  - "idle timeout"
owner: devops-agent
objective: "Resolve compute fleet auto-scaling issues"
context: "ParallelCluster uses Slurm's power-saving plugin and clustermgtd for scaling, NOT EC2 Auto Scaling groups. Nodes launch when jobs are submitted and terminate after ScaledownIdletime. Scaling parameters are in the cluster configuration."
---

## Phase 1 — Triage

MUST:
- Check fleet status: `pcluster describe-compute-fleet --cluster-name <cluster-name>`
- Check node power states: `sinfo -N -l` (idle~ = powered down, alloc = running job)
- Review clustermgtd log for scaling decisions
- Review slurm_resume.log for launch attempts
- Review slurm_suspend.log for termination events

SHOULD:
- Check cluster config for MinCount, MaxCount, ScaledownIdletime
- Verify EC2 launch permissions and quotas
- Check for spot instance interruptions if using spot

MAY:
- Monitor scaling metrics over time
- Check for protected nodes that won't scale down

## Phase 2 — Remediate

MUST:
- If nodes not launching: check slurm_resume.log for EC2 launch errors
- If nodes not terminating: check ScaledownIdletime and slurm_suspend.log
- If scaling too slow: reduce ScaledownIdletime or increase MinCount

SHOULD:
- Set MinCount > 0 for baseline always-on capacity
- Configure appropriate ScaledownIdletime (default 10 min)
- Use multiple instance types for better capacity availability

MAY:
- Implement custom scaling logic with Slurm prolog/epilog scripts
- Set up CloudWatch dashboards for scaling metrics

## Common Issues

- symptoms: "Nodes not scaling up when jobs are submitted"
  diagnosis: "slurm_resume failing to launch EC2 instances."
  resolution: "Check slurm_resume.log. Common: capacity, permissions, or AMI issues."

- symptoms: "Nodes not scaling down after jobs complete"
  diagnosis: "ScaledownIdletime not elapsed or nodes marked as protected."
  resolution: "Wait for idle timeout. Check for protected/reserved nodes."

## Output Format

```yaml
root_cause: "auto_scaling — <specific_cause>"
evidence:
  - type: fleet_status
    content: "<compute fleet state>"
  - type: scaling_logs
    content: "<resume/suspend log entries>"
severity: HIGH
mitigation:
  immediate: "Fix scaling configuration or launch errors"
  long_term: "Implement scaling monitoring and optimization"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — scaling operations are operational
- Service Disruption Risk: MEDIUM — scaling issues affect job throughput
- Compliance Impact: LOW — scaling is an operational concern

## Escalation Conditions
- Auto-scaling fails to add compute nodes when jobs are queued
- Scaling causes cost overruns
- Scale-down terminates nodes with running jobs
- Scaling rate insufficient for workload demands
- Scaling configuration conflicts with scheduler settings

## Data Sensitivity
- Scaling metrics — INTERNAL
- Cost data — INTERNAL
- Instance launch configurations — INTERNAL
- Scaling policies — INTERNAL
- Never expose scaling configurations that reveal workload patterns

## Prohibited Actions
- Do NOT set maximum node count without cost approval
- Do NOT disable auto-scaling on production clusters without a plan
- Do NOT modify scaling parameters during peak job execution
- Do NOT ignore scaling alerts
- Do NOT bypass scaling cooldown periods
- Do NOT set minimum nodes to zero on production clusters

## Phase 3 — Rollback
- Restore previous scaling configuration
- Revert maximum/minimum node count changes
- Re-enable auto-scaling if disabled
- Verify scaling behavior after configuration changes
- Restore scaling cooldown settings
- Document scaling issues and capacity planning updates

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

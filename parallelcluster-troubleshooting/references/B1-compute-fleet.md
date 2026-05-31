---
title: "B1 — Compute Fleet Issues"
description: "Diagnose compute fleet launch and management problems"
status: active
severity: HIGH
triggers:
  - "compute fleet"
  - "compute nodes not launching"
  - "fleet status"
  - "nodes down"
owner: devops-agent
objective: "Resolve compute fleet launch, scaling, and health issues"
context: "Compute fleet nodes are dynamically launched by Slurm's power-saving plugin. Nodes show as idle~ when powered down (normal). Launch failures appear in clustermgtd and slurm_resume logs on the head node."
---

## Phase 1 — Triage

MUST:
- Check fleet status: `pcluster describe-compute-fleet --cluster-name <cluster-name>`
- Check node status from head node: `sinfo -N -l`
- Review clustermgtd log: `pcluster get-cluster-log-events --cluster-name <cluster-name> --log-stream-name ip-<head-ip>.i-<id>.clustermgtd`
- Review resume log: `pcluster get-cluster-log-events --cluster-name <cluster-name> --log-stream-name ip-<head-ip>.i-<id>.slurm_resume`

SHOULD:
- Check for InsufficientInstanceCapacity errors in logs
- Verify instance type availability in the target AZ
- Check EC2 service quotas: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A`

MAY:
- Check spot instance pricing if using spot: `aws ec2 describe-spot-price-history --instance-types <type> --availability-zone <az>`
- Review compute resource configuration in cluster config

## Phase 2 — Remediate

MUST:
- If capacity issue: try different instance types or AZs in the queue config
- If spot interruption: configure on-demand fallback or multiple instance types
- If node health check failing: check clustermgtd logs for specific errors

SHOULD:
- Configure multiple instance types per queue for flexibility
- Set appropriate MinCount for always-on baseline capacity
- Monitor fleet status with CloudWatch

MAY:
- Implement capacity reservations for critical workloads
- Use multiple queues across different AZs

## Common Issues

- symptoms: "Nodes stuck in idle~ state, jobs pending"
  diagnosis: "EC2 launch failures — check slurm_resume.log for specific error."
  resolution: "Fix the launch error (capacity, permissions, AMI). Nodes will retry."

- symptoms: "Nodes launch but go to drain state"
  diagnosis: "Node health check failing — Slurm marks unhealthy nodes as drained."
  resolution: "Check clustermgtd log. Common causes: storage mount failure, network issue."

## Output Format

```yaml
root_cause: "compute_fleet — <specific_cause>"
evidence:
  - type: fleet_status
    content: "<compute fleet state>"
  - type: logs
    content: "<clustermgtd/slurm_resume errors>"
severity: HIGH
mitigation:
  immediate: "Fix launch errors or node health issues"
  long_term: "Configure multi-instance-type queues and capacity monitoring"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — compute fleet management is operational
- Service Disruption Risk: HIGH — fleet issues prevent job execution
- Compliance Impact: LOW — fleet management is an operational concern

## Escalation Conditions
- Compute nodes fail to launch
- Fleet scaling does not respond to job queue demand
- Compute nodes crash repeatedly
- Instance type unavailable in target AZ
- Fleet health checks fail consistently

## Data Sensitivity
- Compute node configurations — INTERNAL
- Fleet scaling metrics — INTERNAL
- Instance health data — INTERNAL
- Job queue statistics — INTERNAL
- Never expose compute node SSH access details

## Prohibited Actions
- Do NOT manually terminate compute nodes with running jobs
- Do NOT change compute fleet instance types without testing
- Do NOT disable fleet health checks
- Do NOT bypass fleet scaling limits without capacity planning
- Do NOT ignore compute node failure patterns
- Do NOT modify fleet configurations during active job execution

## Phase 3 — Rollback
- Restore previous compute fleet configuration
- Re-enable fleet scaling if disabled
- Restart failed compute nodes
- Revert instance type changes
- Verify job execution resumes after fleet recovery
- Document fleet issues and scaling behavior

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

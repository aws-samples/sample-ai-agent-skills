---
title: "A2 — Cluster Update Issues"
description: "Diagnose ParallelCluster update failures and configuration change problems"
status: active
severity: HIGH
triggers:
  - "cluster update failed"
  - "UPDATE_FAILED"
  - "pcluster update"
  - "cannot update"
owner: devops-agent
objective: "Resolve cluster update failures and identify immutable settings"
context: "Some ParallelCluster settings can be updated in place, others require cluster recreation. Update failures may be due to immutable settings, running jobs, or CloudFormation errors."
---

## Phase 1 — Triage

MUST:
- Check cluster status: `pcluster describe-cluster --cluster-name <cluster-name>`
- Dry-run the update: `pcluster update-cluster --cluster-name <cluster-name> --cluster-configuration <new-config> --dryrun true`
- Check stack events: `pcluster get-cluster-stack-events --cluster-name <cluster-name>`
- Verify no jobs are running that block the update: `pcluster ssh --cluster-name <cluster-name> -- squeue`

SHOULD:
- Compare old and new configuration files
- Check if the changed settings are updatable or immutable
- Review update change set from dry-run output

MAY:
- Check CloudFormation change set details
- Review ParallelCluster documentation for update compatibility

## Phase 2 — Remediate

MUST:
- If immutable setting: plan cluster recreation with the new configuration
- If running jobs blocking: wait for jobs to complete or cancel them
- If CloudFormation error: fix the configuration and retry

SHOULD:
- Always run --dryrun before actual update
- Stop compute fleet before major updates: `pcluster update-compute-fleet --cluster-name <cluster-name> --status STOP_REQUESTED`
- Back up data before cluster recreation

MAY:
- Use blue-green deployment for zero-downtime updates
- Implement configuration version control

## Common Issues

- symptoms: "Update blocked — setting is immutable"
  diagnosis: "Attempting to change VPC, subnet, or scheduler configuration."
  resolution: "These require cluster recreation. Back up data and recreate."

- symptoms: "Update fails with compute fleet running"
  diagnosis: "Some updates require compute fleet to be stopped."
  resolution: "Stop compute fleet, apply update, restart fleet."

## Output Format

```yaml
root_cause: "update_failure — <specific_cause>"
evidence:
  - type: dryrun_result
    content: "<update compatibility check>"
  - type: stack_events
    content: "<CloudFormation events>"
severity: HIGH
mitigation:
  immediate: "Fix configuration or plan cluster recreation"
  long_term: "Implement update testing and blue-green deployment"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — updates modify cluster configuration
- Service Disruption Risk: HIGH — failed updates can leave clusters in degraded state
- Compliance Impact: MEDIUM — configuration changes may affect security posture

## Escalation Conditions
- Cluster update fails leaving cluster in UPDATE_FAILED state
- Update causes running jobs to fail
- Update changes break compute fleet scaling
- Rollback after failed update is not possible
- Update involves breaking changes to scheduler configuration

## Data Sensitivity
- Cluster configuration parameters — INTERNAL
- Scheduler configurations — INTERNAL
- Shared storage configurations — INTERNAL
- Network configurations — INTERNAL
- Never expose cluster admin credentials during updates

## Prohibited Actions
- Do NOT update clusters during active HPC job execution without approval
- Do NOT change scheduler type on existing clusters
- Do NOT remove shared storage during update without data migration
- Do NOT update head node instance type without understanding impact
- Do NOT bypass update validation checks
- Do NOT update clusters without a rollback plan

## Phase 3 — Rollback
- If update fails, attempt to restore previous cluster configuration
- Re-create cluster from previous configuration if rollback is not possible
- Restore shared storage configurations
- Verify scheduler functionality after rollback
- Re-submit failed jobs after cluster recovery
- Document update failures and configuration changes

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

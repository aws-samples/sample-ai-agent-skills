---
title: "Z1 — General ParallelCluster Troubleshooting (Catch-All)"
description: "Fallback SOP for ParallelCluster issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown ParallelCluster issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Check cluster status: `pcluster describe-cluster --cluster-name <cluster-name>`
- Check compute fleet: `pcluster describe-compute-fleet --cluster-name <cluster-name>`
- Get stack events: `pcluster get-cluster-stack-events --cluster-name <cluster-name>`
- List log streams: `pcluster list-cluster-log-streams --cluster-name <cluster-name>`
- Check Slurm status (on head node): `sinfo` and `squeue`

SHOULD:
- Review clustermgtd log for errors
- Check CloudTrail for recent changes
- Verify shared storage mounts
- Check head node system health

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Stack creation/update/deletion → Cluster (A1-A3)
- Compute fleet or Slurm → Compute (B1-B2)
- Storage mount or performance → Storage (C1-C2)
- VPC or multi-AZ → Networking (D1-D2)
- Job submission or execution → Jobs (E1-E2)
- Scaling or capacity → Scaling (F1-F2)
- IAM or SSH → Security (G1-G2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- SSH to head node and check all logs in /var/log/parallelcluster/
- Check system logs: dmesg, /var/log/messages
- Verify all services are running: slurmctld, clustermgtd, supervisord
- Check disk space and memory on head node

SHOULD:
- Compare with known-good cluster configuration
- Check AWS Health Dashboard for service events
- Review ParallelCluster release notes for known issues

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<cluster|compute|storage|networking|jobs|scaling|security|unknown>"
investigation_path: "cluster status → fleet → slurm → logs → <domain_classification>"
evidence:
  - type: cluster_status
    content: "<cluster state>"
  - type: slurm_status
    content: "<sinfo/squeue output>"
  - type: logs
    content: "<relevant log entries>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — general troubleshooting uses read-only operations
- Service Disruption Risk: LOW — diagnostic commands do not affect workloads
- Compliance Impact: LOW — standard troubleshooting procedures

## Escalation Conditions
- Issue cannot be resolved using standard troubleshooting procedures
- Cluster enters an unrecoverable state
- Multiple unrelated issues occur simultaneously
- AWS Health Dashboard shows ParallelCluster service event
- Issue requires AWS Support engagement

## Data Sensitivity
- Cluster configuration — INTERNAL
- CloudWatch metrics and logs — INTERNAL
- CloudFormation stack details — INTERNAL
- Error messages — INTERNAL
- Never expose cluster details in external communications

## Prohibited Actions
- Do NOT make changes to production clusters without change management approval
- Do NOT run destructive commands during initial diagnosis
- Do NOT share cluster details in public forums without sanitizing
- Do NOT bypass established escalation procedures
- Do NOT attempt fixes outside ParallelCluster service boundaries
- Do NOT ignore recurring issues

## Phase 3 — Rollback
- Revert any diagnostic changes made during troubleshooting
- Restore cluster to last known good configuration
- Document all troubleshooting steps and outcomes
- Verify cluster health after troubleshooting
- Update runbooks with new findings
- Close out temporary access granted for troubleshooting

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

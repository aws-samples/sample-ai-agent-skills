---
title: "Z1 — General AWS Batch Troubleshooting"
description: "Catch-all systematic investigation for AWS Batch issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "Batch issue"
  - "Batch problem"
  - "Batch not working"
  - "Batch error"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate AWS Batch issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for AWS Batch issues that don't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- Check compute environments: `aws batch describe-compute-environments --query 'computeEnvironments[*].{Name:computeEnvironmentName,State:state,Status:status}'`
- Check job queues: `aws batch describe-job-queues --query 'jobQueues[*].{Name:jobQueueName,State:state,Status:status,Priority:priority}'`
- Check recent failed jobs: `aws batch list-jobs --job-queue <queue-name> --job-status FAILED --max-results 10`
- Check RUNNABLE jobs: `aws batch list-jobs --job-queue <queue-name> --job-status RUNNABLE --max-results 10`
- Check job definitions: `aws batch describe-job-definitions --status ACTIVE --max-results 10`

SHOULD:
- Check CloudTrail for recent Batch changes
- Review CloudWatch metrics for Batch
- Verify IAM roles and permissions
- Check VPC networking configuration

MAY:
- Check AWS Health Dashboard for Batch service issues
- Review ECS cluster associated with CEs
- Check service quotas

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Job issues → A1 (failures), A2 (stuck RUNNABLE), A3 (timeout)
  - Compute → B1 (CE failures), B2 (EC2 vs Fargate), B3 (Spot)
  - Queue → C1 (queue issues), C2 (scheduling)
  - Definition → D1 (definition errors), D2 (container config)
  - Networking → E1 (VPC), E2 (ECR access)
  - Array/Multi → F1 (array jobs), F2 (multi-node)
  - Performance → G1 (throughput), G2 (optimization)
  - Security → H1 (IAM roles), H2 (secrets)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Review recent changes

MAY:
- Set up comprehensive monitoring
- Create post-incident report

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Check compute environments and job queues | GREEN | Read-only investigation |
| Check recent failed and RUNNABLE jobs | GREEN | Read-only investigation |
| Check CloudTrail and CloudWatch | GREEN | Read-only investigation |
| Pivot to specific runbook | YELLOW | Specific runbook may include state-changing actions |
| Set up comprehensive monitoring | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Compute environment runs production batch jobs
- Issue cannot be classified after systematic investigation
- Multiple concurrent issues affect Batch infrastructure
- Issue impacts multiple job queues or compute environments

## Data Sensitivity

- **Classification: MEDIUM**
- CE and queue status reveals Batch infrastructure health
- Job status reveals processing state and failure patterns
- CloudTrail events expose all Batch API calls and configurations
- Job definitions reveal application deployment details

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest making broad configuration changes during general investigation
- **NEVER** suggest disabling compute environments or queues without understanding impact
- **NEVER** suggest cancelling running jobs without confirming with the user

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| General triage is read-only | No rollback needed for Phase 1 investigation |
| Pivot to specific runbook | Follow rollback steps in the specific runbook |

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: ce_status
    content: "<compute environment states>"
  - type: queue_status
    content: "<job queue states>"
  - type: job_status
    content: "<recent job results>"
severity: MEDIUM
general_analysis:
  compute: "healthy | degraded | INVALID"
  queues: "active | disabled"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive monitoring and alerting"
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

---
title: "E1 — Cost Management"
description: "Diagnose unexpected Braket costs and optimize spending"
status: active
severity: MEDIUM
triggers: ["braket cost high", "unexpected charges", "braket billing", "cost optimization"]
owner: devops-agent
objective: "Identify and resolve unexpected Braket costs"
context: "Braket charges per-task and per-shot for QPUs, per-minute for simulators, and standard rates for notebook instances."
---

## Phase 1 — Triage

MUST:
- Review recent tasks and their costs: `aws braket search-quantum-tasks --filters '[{"name":"createdAfter","operator":"EQUAL","values":["<date>"]}]'`
- Check for running hybrid jobs: `aws braket search-jobs --filters '[{"name":"status","operator":"EQUAL","values":["RUNNING"]}]'`
- Review Cost Explorer for Braket charges: `aws ce get-cost-and-usage --time-period Start=<start>,End=<end> --granularity DAILY --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Braket"]}}' --metrics UnblendedCost`

SHOULD:
- Identify which devices generated the most cost
- Check for orphaned notebook instances: `aws sagemaker list-notebook-instances --status-equals InService`
- Review shot counts on QPU tasks — high shot counts multiply per-shot charges

MAY:
- Set up AWS Budgets alerts for Braket spending
- Analyze cost trends over the past month

## Phase 2 — Remediate

MUST:
- Cancel unnecessary running jobs: `aws braket cancel-job --job-arn <arn>`
- Stop unused notebook instances: `aws sagemaker stop-notebook-instance --notebook-instance-name <name>`
- Cancel queued tasks that are no longer needed

SHOULD:
- Use local simulators for development and testing (no cloud charges)
- Reduce shot counts where statistical precision allows
- Choose cost-effective devices for the workload

MAY:
- Implement cost tagging for Braket resources
- Set up automated notebook instance shutdown schedules

## Safety Ratings

```
safety_ratings:
  - "search-quantum-tasks: GREEN — read-only enumeration"
  - "ce get-cost-and-usage: GREEN — read-only billing data"
  - "cancel-job: YELLOW — stops running job, partial charges apply"
  - "stop-notebook-instance: YELLOW — stops compute, may interrupt work"
```

## Escalation Conditions

- Braket charges exceed budget by more than 50%
- Unauthorized tasks or jobs found in the account
- Cost anomalies that cannot be attributed to known workloads
- Charges for devices or regions not used by the team

## Data Sensitivity

- `aws ce get-cost-and-usage`: MEDIUM — reveals spending patterns
- `aws braket search-quantum-tasks`: LOW — returns task metadata
- Cost allocation data: MEDIUM — reveals project spending

## Prohibited Actions

- NEVER cancel running tasks or jobs without confirming they are not critical workloads
- NEVER delete S3 result buckets to reduce storage costs without backing up results
- NEVER disable Braket access at the IAM level as a cost control — use AWS Budgets instead
- NEVER assume all Braket charges are from quantum tasks — notebook instances and S3 storage also contribute

## Phase 3 — Rollback

1. If a job was cancelled in error, resubmit with the same parameters
2. If a notebook instance was stopped, restart it: `aws sagemaker start-notebook-instance --notebook-instance-name <name>`
3. Restore any IAM policy changes made for cost control
4. Document cost optimization measures implemented
5. Verify critical workloads were not disrupted by cost reduction actions

## Common Issues

- symptoms: "Unexpected high Braket charges on the monthly bill"
  diagnosis: "Orphaned notebook instances running 24/7 or high-shot QPU tasks."
  resolution: "Stop unused notebooks. Review task shot counts. Use local simulators for development."

- symptoms: "Charges from a region where no Braket work is being done"
  diagnosis: "Resources may have been created in the wrong region or by unauthorized users."
  resolution: "List all Braket resources across regions. Review CloudTrail for resource creation events."

## Output Format

```yaml
root_cause: "cost_management — <specific_cause>"
evidence:
  - type: cost_analysis
    content: "<cost breakdown by device, task, and resource>"
severity: MEDIUM
mitigation:
  immediate: "Stop unnecessary resources and cancel queued tasks"
  long_term: "Implement budgets, tagging, and automated shutdown"
```

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

---
title: "A2 — Task Stuck in Launching/Queued State"
description: "Diagnose why a DataSync task is stuck and not progressing"
status: active
severity: HIGH
triggers:
  - "task stuck"
  - "task not starting"
  - "task in launching state"
  - "task queued"
  - "execution not progressing"
owner: devops-agent
objective: "Identify why a DataSync task execution is stuck and resolve the blockage"
context: "Tasks can get stuck in QUEUED (agent busy), LAUNCHING (agent connectivity), or PREPARING (scanning large datasets). Each state indicates a different root cause. Agents process one task at a time by default."
---

## Phase 1 — Triage

MUST:
- Check current execution status: `aws datasync describe-task-execution --task-execution-arn <task-execution-arn>`
- Check agent status and connectivity: `aws datasync describe-agent --agent-arn <agent-arn>`
- List all active task executions on the agent: `aws datasync list-task-executions --task-arn <task-arn>`
- Check if other tasks are running on the same agent

SHOULD:
- Verify agent network connectivity (port 443 outbound to DataSync service)
- Check agent VM resource utilization (CPU, memory, disk)
- Review CloudWatch Logs for agent-related errors

MAY:
- Check if VPC endpoint for DataSync is healthy (if using VPC endpoints)
- Verify DNS resolution from agent to DataSync service endpoint

## Phase 2 — Remediate

MUST:
- For QUEUED: Wait for current task to complete or cancel it: `aws datasync cancel-task-execution --task-execution-arn <running-execution-arn>`
- For LAUNCHING: Fix agent connectivity — verify security groups allow port 443 outbound
- For PREPARING: Wait for file scanning to complete (expected for large datasets)

SHOULD:
- Restart agent if it shows as OFFLINE: power cycle the agent VM
- Stagger task schedules to avoid queuing on the same agent

MAY:
- Deploy additional agents to handle concurrent workloads
- Increase agent VM resources for large file count scenarios

## Common Issues

- symptoms: "Task stuck in QUEUED for hours"
  diagnosis: "Another task execution is running on the same agent."
  resolution: "Wait for the running task or cancel it. Agents handle one task at a time."

- symptoms: "Task stuck in LAUNCHING"
  diagnosis: "Agent cannot communicate with DataSync service."
  resolution: "Check agent status. Verify port 443 outbound. Check VPC endpoint if applicable."

- symptoms: "Task stuck in PREPARING for a long time"
  diagnosis: "Source or destination has millions of files; scanning takes time."
  resolution: "This is expected for large datasets. Monitor progress via CloudWatch metrics."

## Output Format

```yaml
root_cause: "task_stuck — <specific_state_and_cause>"
evidence:
  - type: execution_state
    content: "<current execution state>"
  - type: agent_status
    content: "<agent connectivity status>"
  - type: concurrent_tasks
    content: "<other running tasks on agent>"
severity: HIGH
mitigation:
  immediate: "Resolve the specific blockage based on stuck state"
  long_term: "Distribute tasks across multiple agents; stagger schedules"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Stuck tasks may hold agent resources needed for other transfers |
| Blast Radius | MEDIUM — Stuck tasks block other tasks on the same agent |
| Rollback Complexity | LOW — Cancel stuck execution and restart |
| Compliance Scope | MEDIUM — Stuck tasks may delay compliance-required data transfers |

## Escalation Conditions

- Escalate to operations if stuck tasks block time-sensitive data transfers
- Escalate to network team if LAUNCHING state indicates connectivity issues
- Escalate to AWS Support if tasks remain stuck after agent restart

## Data Sensitivity

- **Agent connectivity details** reveal network architecture — handle appropriately
- **Task execution metadata** is operational data

## PROHIBITED Actions

- Do NOT cancel running tasks without understanding the impact on data consistency
- Do NOT power-cycle the agent VM without checking for other active tasks
- Do NOT ignore stuck tasks — they indicate underlying infrastructure issues

## Phase 3 — Rollback

- Cancel the stuck execution: `aws datasync cancel-task-execution --task-execution-arn <arn>`
- Restart the agent VM if it is unresponsive
- Start a new task execution after the agent returns to ONLINE status
- Verify no data corruption from the cancelled execution

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

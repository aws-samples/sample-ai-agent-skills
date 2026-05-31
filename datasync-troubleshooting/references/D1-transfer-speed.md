---
title: "D1 — Slow Transfer Speed"
description: "Diagnose slow DataSync transfer performance"
status: active
severity: MEDIUM
triggers:
  - "slow transfer"
  - "transfer speed"
  - "low throughput"
  - "transfer taking too long"
owner: devops-agent
objective: "Identify bottlenecks causing slow DataSync transfers and optimize performance"
context: "DataSync transfer speed depends on network bandwidth, agent resources, source/destination I/O, file size distribution, and task configuration. Small files transfer slower per-byte than large files due to per-file overhead. Agent VM sizing directly impacts performance."
---

## Phase 1 — Triage

MUST:
- Check transfer metrics: `aws cloudwatch get-metric-statistics --namespace AWS/DataSync --metric-name BytesTransferred --dimensions Name=TaskId,Value=<task-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check agent VM resources: CPU, memory, disk I/O utilization
- Verify network bandwidth between agent and source/destination
- Check task execution details: `aws datasync describe-task-execution --task-execution-arn <execution-arn>`

SHOULD:
- Check file size distribution (many small files = slower per-byte throughput)
- Verify bandwidth throttling is not set too low: `aws datasync describe-task --task-arn <task-arn> --query 'Options.BytesPerSecond'`
- Check if verification mode is adding overhead (POINT_IN_TIME_CONSISTENT is slower)

MAY:
- Compare transfer speed with expected network bandwidth
- Check for network congestion or packet loss on the transfer path

## Phase 2 — Remediate

MUST:
- Remove or increase bandwidth throttling if set too low
- Ensure agent VM meets minimum requirements (4 vCPUs, 16 GB RAM)
- Verify network path has sufficient bandwidth

SHOULD:
- Increase agent VM resources for large transfers: 8+ vCPUs, 32+ GB RAM
- Use ONLY_FILES_TRANSFERRED verification instead of POINT_IN_TIME_CONSISTENT
- Reduce file count per task using filters if possible

MAY:
- Deploy agent closer to source storage to reduce latency
- Use multiple tasks with different filters to parallelize transfers
- Consider AWS Direct Connect for on-premises to AWS transfers

## Common Issues

- symptoms: "Transfer speed much lower than network bandwidth"
  diagnosis: "Agent VM undersized or many small files causing per-file overhead."
  resolution: "Increase agent resources. For small files, expect lower per-byte throughput."

- symptoms: "Transfer speed drops during execution"
  diagnosis: "Bandwidth throttling, network congestion, or source I/O bottleneck."
  resolution: "Check BytesPerSecond setting. Monitor network and source storage I/O."

- symptoms: "PREPARING phase takes longer than TRANSFERRING"
  diagnosis: "Large number of files at source/destination requires extensive scanning."
  resolution: "Expected for millions of files. Use filters to reduce scope if possible."

## Output Format

```yaml
root_cause: "slow_transfer — <specific_cause>"
evidence:
  - type: transfer_metrics
    content: "<bytes transferred over time>"
  - type: agent_resources
    content: "<agent VM CPU, memory, disk utilization>"
  - type: task_config
    content: "<bandwidth throttling and verification settings>"
severity: MEDIUM
mitigation:
  immediate: "Remove bottleneck (throttling, agent resources, network)"
  long_term: "Right-size agent VM and optimize task configuration"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Performance analysis uses transfer metrics, not data content |
| Blast Radius | MEDIUM — Performance changes (agent sizing, throttling) affect all tasks on the agent |
| Rollback Complexity | LOW — Revert throttling or agent configuration changes |
| Compliance Scope | MEDIUM — Transfer speed affects SLA compliance for data delivery |

## Escalation Conditions

- Escalate to network team if bandwidth is the bottleneck
- Escalate to operations if slow transfers miss data delivery SLAs
- Escalate to infrastructure team if agent VM needs resource upgrades

## Data Sensitivity

- **Transfer metrics** are operational data — low sensitivity
- **Agent VM resource utilization** is infrastructure data

## PROHIBITED Actions

- Do NOT remove bandwidth throttling without coordinating with network team
- Do NOT resize agent VMs during active task executions
- Do NOT change verification mode to NONE for compliance-required transfers just to improve speed

## Phase 3 — Rollback

- Restore previous bandwidth throttling settings
- Revert agent VM resource changes if they caused issues
- Restore verification mode to previous setting

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

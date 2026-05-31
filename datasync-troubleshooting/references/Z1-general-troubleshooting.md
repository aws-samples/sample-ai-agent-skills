---
title: "Z1 — General DataSync Troubleshooting"
description: "Catch-all systematic investigation for DataSync issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "DataSync issue"
  - "DataSync problem"
  - "DataSync error"
  - "transfer not working"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate DataSync issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for DataSync issues that don't clearly match a specific runbook. It covers task health, agent status, location validation, transfer metrics, and systematic elimination of common causes. Use this as a starting point when the issue category is unclear, then pivot to a specific runbook once the problem area is identified."
---

## Phase 1 — Triage

MUST:
- List all tasks and their status: `aws datasync list-tasks --query 'Tasks[*].{Arn:TaskArn,Status:Status,Name:Name}'`
- Check all agents: `aws datasync list-agents --query 'Agents[*].{Arn:AgentArn,Name:Name,Status:Status}'`
- Check recent task executions: `aws datasync list-task-executions --task-arn <task-arn> --query 'TaskExecutions[*].{Arn:TaskExecutionArn,Status:Status}'`
- Describe the failing task: `aws datasync describe-task --task-arn <task-arn>`
- Describe the latest execution: `aws datasync describe-task-execution --task-execution-arn <execution-arn>`

SHOULD:
- Check CloudTrail for DataSync API errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=datasync.amazonaws.com --max-results 20`
- Check CloudWatch Logs for task execution details: `aws logs filter-log-events --log-group-name /aws/datasync --filter-pattern "ERROR"`
- Verify source and destination locations are accessible
- Check CloudWatch metrics: BytesTransferred, FilesTransferred

MAY:
- Check AWS Health Dashboard for DataSync service issues
- Review recent configuration changes
- Check IAM permissions and role configurations

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Task issues → A1 (failures), A2 (stuck), A3 (verification)
  - Agent issues → B1 (deployment), B2 (connectivity)
  - Location issues → C1 (S3), C2 (EFS/FSx), C3 (NFS/SMB)
  - Performance → D1 (speed), D2 (throttling)
  - Filtering → E1 (filters), E2 (metadata)
  - Security → F1 (IAM), F2 (encryption)
  - Scheduling → G1 (schedule), G2 (incremental)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Review recent changes that may have caused the issue
- Verify the fix resolves the issue without introducing new problems

MAY:
- Set up comprehensive monitoring to prevent recurrence
- Create a post-incident report for significant issues

## Common Issues

- symptoms: "DataSync not working — no specific error"
  diagnosis: "Multiple possible causes: agent, location, permissions, network."
  resolution: "Follow triage steps: agent status → task status → location → permissions."

- symptoms: "Task was working but stopped"
  diagnosis: "Recent change to agent, network, permissions, or source/destination."
  resolution: "Check CloudTrail for recent changes. Compare current config with last successful run."

- symptoms: "Some files transfer but others don't"
  diagnosis: "Filter configuration, permission issues on specific files, or file locking."
  resolution: "Check filters (E1). Verify permissions on failing files. Check for file locks."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: task_status
    content: "<task and execution status>"
  - type: agent_status
    content: "<agent connectivity>"
  - type: location_config
    content: "<source and destination configuration>"
  - type: error_details
    content: "<specific errors from logs or execution results>"
severity: MEDIUM
general_analysis:
  task_state: "available | running | unavailable"
  agent_state: "online | offline"
  location_valid: "yes | no"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive monitoring and alerting"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — General troubleshooting may involve examining various DataSync resources |
| Blast Radius | LOW — Diagnostic commands are read-only |
| Rollback Complexity | LOW — No changes during triage |
| Compliance Scope | MEDIUM — Ensure troubleshooting does not expose sensitive data paths |

## Escalation Conditions

- Escalate to the specific runbook owner once the problem domain is identified
- Escalate to security if troubleshooting reveals unauthorized data transfers
- Escalate to AWS Support if systematic issues affect multiple DataSync components

## Data Sensitivity

- **Task and location configurations** may reveal data paths and storage architecture
- **CloudTrail events** contain API call details
- **CloudWatch Logs** may contain file paths from transfer operations

## PROHIBITED Actions

- Do NOT modify task configurations during initial triage
- Do NOT cancel running tasks without understanding the data consistency impact
- Do NOT share data transfer paths or storage architecture details outside operations team

## Phase 3 — Rollback

- No rollback needed for diagnostic operations
- Revert any configuration changes made during troubleshooting
- Route to the appropriate specific runbook for remediation

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

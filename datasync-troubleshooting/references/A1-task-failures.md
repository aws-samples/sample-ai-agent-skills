---
title: "A1 — Task Execution Failures"
description: "Diagnose why a DataSync task execution is failing"
status: active
severity: HIGH
triggers:
  - "task execution failed"
  - "task error"
  - "DataSync task failure"
  - "execution status ERROR"
owner: devops-agent
objective: "Identify why a DataSync task execution is failing and resolve the issue"
context: "Task execution failures can result from agent connectivity issues, location access problems, IAM permission errors, network timeouts, source/destination unavailability, or resource limits. The task execution result contains error details that guide diagnosis."
---

## Phase 1 — Triage

MUST:
- Check task execution status and result: `aws datasync describe-task-execution --task-execution-arn <task-execution-arn>`
- Check agent status: `aws datasync list-agents --query 'Agents[*].{Arn:AgentArn,Status:Status}'`
- Check task configuration: `aws datasync describe-task --task-arn <task-arn>`
- Check source and destination locations: `aws datasync describe-location-s3 --location-arn <source-location-arn>` or appropriate describe-location-* command

SHOULD:
- Review CloudWatch Logs for task execution: `aws logs filter-log-events --log-group-name /aws/datasync --filter-pattern "<task-id>"`
- Check CloudTrail for recent errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=datasync.amazonaws.com --max-results 20`
- Verify IAM role permissions for S3 locations: `aws iam get-role --role-name <datasync-s3-role>`

MAY:
- Check CloudWatch metrics for the task: BytesTransferred, FilesTransferred
- Verify network connectivity from agent to source/destination

## Phase 2 — Remediate

MUST:
- Fix the specific error identified in the task execution result
- Ensure agent is ONLINE before retrying
- Verify source and destination are accessible

SHOULD:
- Start a new task execution after fixing the issue: `aws datasync start-task-execution --task-arn <task-arn>`
- Enable CloudWatch logging for future executions: `aws datasync update-task --task-arn <task-arn> --cloud-watch-log-group-arn <log-group-arn>`

MAY:
- Adjust task options (verification, overwrite mode) if appropriate
- Set up CloudWatch alarms for task execution failures

## Common Issues

- symptoms: "Task fails immediately after starting"
  diagnosis: "Agent is offline or cannot reach source/destination."
  resolution: "Check agent status. Verify network connectivity. Restart agent if needed."

- symptoms: "Permission denied errors in execution result"
  diagnosis: "IAM role lacks required permissions for S3/EFS/FSx access."
  resolution: "Update IAM role with required permissions. For S3: s3:GetObject, s3:PutObject, s3:ListBucket."

- symptoms: "Task fails during TRANSFERRING phase"
  diagnosis: "Network interruption, source file changes, or destination write failures."
  resolution: "Check network stability. Verify source files are not locked. Check destination capacity."

## Output Format

```yaml
root_cause: "task_execution_failure — <specific_cause>"
evidence:
  - type: execution_status
    content: "<task execution describe output>"
  - type: agent_status
    content: "<agent status>"
  - type: error_details
    content: "<error from execution result>"
severity: HIGH
mitigation:
  immediate: "Fix the identified blocker and retry execution"
  long_term: "Enable CloudWatch logging and set up failure alerts"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Task failures may involve data in transit between storage systems |
| Blast Radius | MEDIUM — Failed tasks leave data partially transferred; retries may cause duplicates |
| Rollback Complexity | MEDIUM — Partial transfers may need cleanup at destination |
| Compliance Scope | HIGH — Data transfer integrity is critical for compliance |

## Escalation Conditions

- Escalate to data owners if task failures leave data in an inconsistent state
- Escalate to security if permission errors indicate IAM misconfiguration
- Escalate to AWS Support if tasks fail with internal service errors

## Data Sensitivity

- **Transferred data** may contain sensitive files — ensure encryption in transit and at rest
- **IAM roles** for S3 locations have broad data access — audit regularly
- **CloudWatch Logs** may contain file paths — restrict access

## PROHIBITED Actions

- Do NOT retry failed tasks without understanding the root cause
- Do NOT modify IAM roles with overly broad permissions to fix access errors
- Do NOT ignore partial transfer states — verify data consistency at destination
- Do NOT disable CloudWatch logging for task executions

## Phase 3 — Rollback

- If partial data was transferred, verify consistency at destination before retrying
- Clean up incomplete files at destination if the task failed mid-transfer
- Restore previous task configuration if settings were changed during troubleshooting
- Retry the task execution after fixing the root cause

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

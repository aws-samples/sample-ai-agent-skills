---
title: "B1 — Job Execution Failures"
description: "Diagnose IoT job execution failures on target devices"
status: active
severity: HIGH
triggers: ["job execution failed", "device not executing job", "job status FAILED", "job document error"]
owner: devops-agent
objective: "Resolve job execution failures and ensure jobs complete on target devices"
context: "IoT jobs send remote operations to devices. Execution status is per-device. Failures can be caused by job document errors, device connectivity, or device-side processing issues."
---

## Phase 1 — Triage

MUST:
- Describe the job: `aws iot describe-job --job-id <job-id> --query '{Status:job.status,Targets:job.targets,TargetSelection:job.targetSelection}'`
- List failed executions: `aws iot list-job-executions-for-job --job-id <job-id> --status FAILED --query 'executionSummaries[*].{ThingArn:thingArn,Status:jobExecutionSummary.status}'`
- Get execution details for a failed device: `aws iot describe-job-execution --job-id <job-id> --thing-name <thing-name> --query '{Status:execution.status,StatusDetails:execution.statusDetails,StartedAt:execution.startedAt}'`

SHOULD:
- Check the job document for syntax errors: `aws iot get-job-document --job-id <job-id>`
- Verify target thing group membership: `aws iot list-things-in-thing-group --thing-group-name <group>`
- Check device connectivity: `aws iot describe-thing --thing-name <name> --query '{Connectivity:connectivity}'` (requires fleet indexing)

MAY:
- Review CloudWatch metrics for job: `aws cloudwatch get-metric-statistics --namespace AWS/IoT --metric-name JobsFailed --dimensions Name=JobId,Value=<job-id> --start-time <time> --end-time <time> --period 300 --statistics Sum`
- Check device MQTT connection status via IoT Core logs

## Phase 2 — Remediate

MUST:
- Fix job document JSON syntax and ensure device can parse it
- Verify target devices are connected and subscribed to job topics ($aws/things/<name>/jobs/*)
- Check device-side job agent is running and processing job notifications

SHOULD:
- Retry failed executions if the issue was transient: configure RetryCriteria on the job
- Verify rollout configuration is not throttling execution delivery
- Check abort criteria thresholds are not prematurely stopping the job

MAY:
- Create a new job targeting only failed devices with corrected job document
- Enable detailed IoT Core logging for job-related MQTT messages

## Safety Ratings

```
safety_ratings:
  - "describe-job: GREEN — read-only inspection"
  - "list-job-executions-for-job: GREEN — read-only enumeration"
  - "get-job-document: GREEN — read-only, but document may contain sensitive URLs"
  - "cancel-job: RED — stops job execution across all target devices"
  - "delete-job: RED — permanently removes job and execution history"
```

## Escalation Conditions

- More than 30% of target devices report FAILED status
- Job failures cause devices to enter an unrecoverable state
- Job document contains credentials or sensitive URLs that may be exposed
- Continuous job failures indicate a systemic device firmware issue

## Data Sensitivity

- `aws iot describe-job`: LOW — returns job metadata
- `aws iot get-job-document`: MEDIUM — may contain pre-signed URLs, configuration data
- `aws iot describe-job-execution`: LOW — returns execution status and details
- Device-side logs: HIGH — may contain device-specific operational data

## Prohibited Actions

- NEVER force-cancel a job with devices mid-execution without assessing device state impact
- NEVER delete a job to hide failures — preserve execution history for analysis
- NEVER modify a job document for an in-progress job (create a new job instead)
- NEVER assume all devices failed for the same reason — check individual execution details

## Phase 3 — Rollback

1. Cancel the job if it should not continue: `aws iot cancel-job --job-id <job-id>` (without --force to let in-progress complete)
2. If devices are in a bad state, create a remediation job to restore previous configuration
3. If job was targeting wrong group, cancel and create new job with correct targets
4. Document failed device list for follow-up remediation
5. Verify device fleet health after rollback by checking shadow states

## Common Issues

- symptoms: "Job execution stays in QUEUED and never moves to IN_PROGRESS"
  diagnosis: "Device is not connected or not subscribed to job notification topics."
  resolution: "Verify device MQTT connection. Ensure device job agent subscribes to $aws/things/<name>/jobs/notify-next."

- symptoms: "Job execution moves to FAILED immediately with statusDetails showing parse error"
  diagnosis: "Job document JSON is malformed or contains fields the device agent cannot process."
  resolution: "Validate job document JSON. Test with a single device before fleet-wide deployment."

## Output Format

```yaml
root_cause: "job_execution — <specific_cause>"
evidence:
  - type: execution_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix job configuration and retry"
  long_term: "Implement job validation and staged rollout"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"

---
title: "A1 — Quantum Task Failures"
description: "Diagnose quantum task execution failures on simulators and QPU devices"
status: active
severity: HIGH
triggers: ["quantum task failed", "task FAILED status", "braket task error"]
owner: devops-agent
objective: "Resolve quantum task failures and identify root causes"
context: "Quantum tasks can fail due to circuit errors, device constraints, IAM permissions, or S3 output issues."
---

## Phase 1 — Triage

MUST:
- Get task details: `aws braket get-quantum-task --quantum-task-arn <arn>`
- Check failureReason field in task response
- Verify IAM role has braket:CreateQuantumTask and s3:PutObject permissions

SHOULD:
- Check device status: `aws braket get-device --device-arn <device-arn>`
- Verify S3 output bucket exists and is accessible
- Review recent failed tasks: `aws braket search-quantum-tasks --filters '[{"name":"status","operator":"EQUAL","values":["FAILED"]}]'`

MAY:
- Check CloudTrail for Braket API errors
- Review CloudWatch metrics for the device

## Phase 2 — Remediate

MUST:
- Fix identified root cause (circuit, permissions, or device issue)
- Validate circuit against device constraints before resubmitting
- Ensure S3 output location is correctly configured

SHOULD:
- Test circuit on local simulator before submitting to QPU
- Verify device supports all gates used in the circuit

MAY:
- Implement retry logic with exponential backoff for transient failures
- Set up CloudWatch alarms for task failure rates

## Safety Ratings

```
safety_ratings:
  - "get-quantum-task: GREEN — read-only task status retrieval"
  - "search-quantum-tasks: GREEN — read-only task enumeration"
  - "get-device: GREEN — read-only device info"
  - "create-quantum-task (resubmit): YELLOW — incurs per-task and per-shot charges"
```

## Escalation Conditions

- Tasks fail consistently across multiple devices with the same circuit
- failureReason indicates internal service error not related to user input
- IAM permissions are correct but tasks still fail with access denied
- Device shows ONLINE but all tasks fail immediately

## Data Sensitivity

- `aws braket get-quantum-task`: LOW — returns task metadata and status
- `aws braket search-quantum-tasks`: LOW — returns task ARNs and statuses
- `aws s3 ls s3://<bucket>/`: MEDIUM — reveals output bucket structure and task IDs

## Prohibited Actions

- NEVER cancel a RUNNING task without user confirmation — QPU time may already be consumed
- NEVER resubmit tasks in a loop without fixing the root cause — each submission incurs charges
- NEVER modify S3 bucket policies without understanding the impact on other Braket tasks
- NEVER assume a circuit is valid on a QPU because it works on a simulator

## Phase 3 — Rollback

1. If a task was submitted with incorrect parameters, cancel it if still in QUEUED status: `aws braket cancel-quantum-task --quantum-task-arn <arn>`
2. If S3 bucket permissions were modified, restore original bucket policy
3. If IAM role was changed, revert to the previous role configuration
4. Verify no orphaned resources (running tasks, active jobs) remain from troubleshooting
5. Document the failure pattern and resolution for future reference

## Common Issues

- symptoms: "Task fails with 'Could not find device' error"
  diagnosis: "Device ARN is incorrect or device is not available in the specified region."
  resolution: "Verify device ARN format and regional availability with aws braket search-devices."

- symptoms: "Task fails with S3 access denied"
  diagnosis: "IAM role lacks s3:PutObject permission on the output bucket."
  resolution: "Update IAM role policy to grant s3:PutObject on the Braket output bucket."

## Output Format

```yaml
root_cause: "quantum_task_failure — <specific_cause>"
evidence:
  - type: task_status
    content: "<failureReason from get-quantum-task>"
severity: HIGH
mitigation:
  immediate: "Fix identified issue and resubmit task"
  long_term: "Implement pre-submission validation and monitoring"
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

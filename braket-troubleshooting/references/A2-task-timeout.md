---
title: "A2 — Task Timeout"
description: "Diagnose quantum task timeout and long queue wait issues"
status: active
severity: MEDIUM
triggers: ["task timeout", "task stuck in queue", "braket task taking too long"]
owner: devops-agent
objective: "Resolve quantum task timeout and excessive wait time issues"
context: "QPU tasks queue during availability windows. Simulators may timeout on complex circuits."
---

## Phase 1 — Triage

MUST:
- Check task status and timestamps: `aws braket get-quantum-task --quantum-task-arn <arn>`
- Verify device availability: `aws braket get-device --device-arn <device-arn>`
- Check if device is in an availability window (deviceStatus field)

SHOULD:
- Compare task creation time vs current time to determine wait duration
- Check queue depth by searching for QUEUED tasks: `aws braket search-quantum-tasks --filters '[{"name":"status","operator":"EQUAL","values":["QUEUED"]}]'`

MAY:
- Check AWS Health Dashboard for Braket service issues
- Review CloudWatch metrics for device queue times

## Phase 2 — Remediate

MUST:
- If device is OFFLINE, wait for next availability window or switch to available device
- For simulator timeouts, reduce circuit complexity (fewer qubits, lower depth)
- Verify task has not exceeded the maximum allowed runtime

SHOULD:
- Consider using a different device with shorter queue times
- For TN1 simulator, ensure circuit structure is compatible with tensor network simulation

MAY:
- Implement task timeout monitoring with CloudWatch
- Use Braket SDK polling with configurable timeout

## Safety Ratings

```
safety_ratings:
  - "get-quantum-task: GREEN — read-only status check"
  - "get-device: GREEN — read-only device info"
  - "cancel-quantum-task: YELLOW — cancels queued task, may lose queue position"
  - "search-quantum-tasks: GREEN — read-only enumeration"
```

## Escalation Conditions

- Tasks remain in QUEUED status for more than 24 hours during a device availability window
- Simulator tasks timeout consistently on circuits within documented limits
- Device shows ONLINE but no tasks are being processed
- Multiple users report queue delays on the same device

## Data Sensitivity

- `aws braket get-quantum-task`: LOW — returns task metadata and timing
- `aws braket get-device`: LOW — returns device status and capabilities
- `aws braket search-quantum-tasks`: LOW — returns task ARNs and statuses

## Prohibited Actions

- NEVER cancel a RUNNING task to "speed things up" — the task is actively executing
- NEVER assume a device is broken because of long queue times — availability windows are normal
- NEVER recommend switching QPU providers without considering circuit compatibility
- NEVER set unrealistic timeout expectations for QPU tasks

## Phase 3 — Rollback

1. If a task was cancelled while queued, resubmit when device is available: `aws braket cancel-quantum-task --quantum-task-arn <arn>`
2. If circuit was simplified for timeout resolution, document original circuit for later execution
3. If device was switched, verify results are comparable between devices
4. Restore any modified SDK timeout configurations to defaults
5. Document queue time observations for capacity planning

## Common Issues

- symptoms: "Task stuck in QUEUED status for hours"
  diagnosis: "QPU device is outside its availability window."
  resolution: "Check device availability windows with get-device. Wait for next window or use a simulator."

- symptoms: "Simulator task times out on a 30+ qubit circuit"
  diagnosis: "SV1 state vector simulation is exponential in qubit count."
  resolution: "Reduce qubit count, use TN1 for structured circuits, or simplify circuit depth."

## Output Format

```yaml
root_cause: "task_timeout — <specific_cause>"
evidence:
  - type: task_timing
    content: "<creation time, status, device availability>"
severity: MEDIUM
mitigation:
  immediate: "Address queue or complexity issue"
  long_term: "Implement timeout monitoring and device selection strategy"
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

---
title: "B2 — Hardware Device Issues"
description: "Diagnose QPU hardware device problems including availability and connectivity"
status: active
severity: HIGH
triggers: ["QPU not available", "hardware device error", "device offline", "IonQ error", "Rigetti error"]
owner: devops-agent
objective: "Resolve QPU hardware device access and execution issues"
context: "QPU devices are provided by third-party vendors (IonQ, Rigetti, IQM, QuEra) with varying availability and capabilities."
---

## Phase 1 — Triage

MUST:
- Check device status: `aws braket get-device --device-arn <device-arn>`
- List available QPU devices: `aws braket search-devices --filters '[{"name":"deviceType","operator":"EQUAL","values":["QPU"]}]'`
- Verify device is in ONLINE status and within availability window

SHOULD:
- Check device capabilities for gate set and qubit connectivity
- Verify circuit compatibility with the specific QPU
- Review recent task history on the device

MAY:
- Check AWS Health Dashboard for device-specific advisories
- Compare device availability across regions

## Phase 2 — Remediate

MUST:
- If device is OFFLINE, wait for next availability window or select alternative device
- Transpile circuit to device-native gate set if gate errors occur
- Verify qubit mapping respects device connectivity constraints

SHOULD:
- Use Braket SDK device.properties to check constraints programmatically
- Test circuit on device-specific simulator before QPU submission

MAY:
- Contact AWS Support for persistent device issues
- Implement device failover logic in application code

## Safety Ratings

```
safety_ratings:
  - "get-device: GREEN — read-only device status"
  - "search-devices: GREEN — read-only device enumeration"
  - "create-quantum-task on QPU: YELLOW — incurs per-task and per-shot charges"
  - "cancel-quantum-task: YELLOW — cancels queued task"
```

## Escalation Conditions

- Device shows ONLINE but consistently fails to execute tasks
- Device availability windows are significantly shorter than documented
- Hardware errors occur on circuits that previously executed successfully
- Device calibration data shows significant degradation

## Data Sensitivity

- `aws braket get-device`: LOW — returns device metadata and capabilities
- `aws braket search-devices`: LOW — returns device ARNs and statuses
- QPU execution results: LOW — quantum measurement outcomes

## Prohibited Actions

- NEVER assume QPU results will match simulator results exactly — hardware noise is expected
- NEVER submit high-shot-count tasks to a QPU without cost estimation
- NEVER claim a specific QPU device will be available at a specific time
- NEVER modify circuit qubit mapping without understanding device topology

## Phase 3 — Rollback

1. Cancel queued tasks on unavailable devices: `aws braket cancel-quantum-task --quantum-task-arn <arn>`
2. If circuit was transpiled for a specific device, retain original circuit definition
3. Revert any application code changes made for device-specific workarounds
4. Document device-specific issues and constraints encountered
5. Verify no tasks are left in QUEUED state on offline devices

## Common Issues

- symptoms: "Device ARN returns 'device not found' error"
  diagnosis: "Device may have been retired or ARN format is incorrect."
  resolution: "List current devices with search-devices. Verify ARN format: arn:aws:braket:<region>::device/qpu/<provider>/<name>."

- symptoms: "Task fails with 'qubit connectivity' error on Rigetti device"
  diagnosis: "Circuit uses two-qubit gates on non-connected qubits in the device topology."
  resolution: "Use Braket SDK automatic qubit mapping or manually map to connected qubits."

## Output Format

```yaml
root_cause: "hardware_device — <specific_cause>"
evidence:
  - type: device_status
    content: "<device status, availability, error details>"
severity: HIGH
mitigation:
  immediate: "Resolve device access or compatibility issue"
  long_term: "Implement device-aware circuit compilation and failover"
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

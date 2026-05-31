---
title: "C1 — Circuit Validation"
description: "Diagnose quantum circuit validation failures before task submission"
status: active
severity: MEDIUM
triggers: ["circuit validation error", "invalid circuit", "unsupported gate", "qubit count exceeded"]
owner: devops-agent
objective: "Resolve circuit validation errors and ensure device compatibility"
context: "Circuits must conform to device-specific constraints including gate sets, qubit counts, connectivity, and depth limits."
---

## Phase 1 — Triage

MUST:
- Identify the target device and its constraints: `aws braket get-device --device-arn <device-arn>`
- Check circuit qubit count against device maximum
- Verify all gates in the circuit are in the device's supported gate set

SHOULD:
- Check qubit connectivity map for two-qubit gate compatibility
- Verify circuit depth is within device limits
- Review Braket SDK version for circuit compilation support

MAY:
- Test circuit on local simulator to verify logical correctness
- Check if SDK automatic transpilation can resolve gate incompatibilities

## Phase 2 — Remediate

MUST:
- Replace unsupported gates with device-native equivalents
- Reduce qubit count to fit within device limits
- Ensure two-qubit gates respect device connectivity

SHOULD:
- Use Braket SDK verbatim=False to enable automatic transpilation
- Decompose complex gates into native gate sequences
- Optimize circuit depth through gate cancellation and merging

MAY:
- Implement circuit optimization passes before submission
- Use device-specific circuit templates for common algorithms

## Safety Ratings

```
safety_ratings:
  - "get-device: GREEN — read-only device capabilities"
  - "local simulator validation: GREEN — no cloud charges"
  - "create-quantum-task (test submission): YELLOW — incurs charges"
  - "circuit modification: GREEN — local code changes only"
```

## Escalation Conditions

- Circuit cannot be transpiled to any available device's native gate set
- Device capabilities documentation contradicts actual device behavior
- SDK transpilation produces incorrect circuit transformations
- Circuit validation passes but task still fails on device

## Data Sensitivity

- `aws braket get-device`: LOW — returns device capabilities
- Circuit definitions: LOW — quantum algorithm structure
- Transpilation results: LOW — modified circuit structure

## Prohibited Actions

- NEVER submit a circuit to a QPU without validating against device constraints
- NEVER assume gate decompositions preserve circuit semantics without verification
- NEVER ignore connectivity constraints — invalid qubit pairs cause task failures
- NEVER claim a circuit is "optimized" without measuring gate count and depth reduction

## Phase 3 — Rollback

1. Restore original circuit definition if transpilation changed the algorithm
2. Revert SDK version if an upgrade caused new validation errors
3. Document device-specific constraints that caused validation failures
4. If circuit was simplified, retain the original for future device compatibility
5. Verify transpiled circuit produces equivalent results on simulator

## Common Issues

- symptoms: "Circuit uses CZ gate but device only supports CNOT"
  diagnosis: "Target device native gate set does not include CZ."
  resolution: "Use verbatim=False in Braket SDK to enable automatic gate transpilation, or manually decompose CZ into supported gates."

- symptoms: "Circuit exceeds maximum qubit count for device"
  diagnosis: "Circuit requires more qubits than the target device supports."
  resolution: "Reduce circuit qubit count, use qubit reuse techniques, or select a device with more qubits."

## Output Format

```yaml
root_cause: "circuit_validation — <specific_cause>"
evidence:
  - type: validation_error
    content: "<gate, qubit, or depth constraint violated>"
severity: MEDIUM
mitigation:
  immediate: "Fix circuit to conform to device constraints"
  long_term: "Implement device-aware circuit compilation pipeline"
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

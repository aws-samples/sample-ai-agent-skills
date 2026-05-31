---
title: "B1 — Simulator Errors"
description: "Diagnose Amazon Braket managed simulator failures"
status: active
severity: MEDIUM
triggers: ["simulator error", "SV1 failed", "DM1 error", "TN1 failure"]
owner: devops-agent
objective: "Resolve simulator-specific execution errors"
context: "Braket offers three managed simulators: SV1 (state vector), DM1 (density matrix), TN1 (tensor network), each with different constraints."
---

## Phase 1 — Triage

MUST:
- Get task details: `aws braket get-quantum-task --quantum-task-arn <arn>`
- Identify which simulator was used (SV1, DM1, TN1)
- Check failureReason for simulator-specific error messages

SHOULD:
- Verify circuit qubit count is within simulator limits (SV1: 34, DM1: 17, TN1: 50)
- Check if circuit uses operations supported by the simulator
- Review task input parameters for correctness

MAY:
- Test the same circuit on a different simulator for comparison
- Check CloudWatch for Braket simulator metrics

## Phase 2 — Remediate

MUST:
- Reduce qubit count if exceeding simulator limits
- Remove unsupported operations for the target simulator
- For TN1, ensure circuit structure is amenable to tensor network contraction

SHOULD:
- Use DM1 for noise simulation circuits (density matrix supports noise channels)
- Use SV1 for general-purpose simulation up to 34 qubits
- Use TN1 for sparse/structured circuits up to 50 qubits

MAY:
- Use local simulator for rapid iteration before cloud submission
- Profile circuit complexity to estimate simulation time

## Safety Ratings

```
safety_ratings:
  - "get-quantum-task: GREEN — read-only task retrieval"
  - "get-device: GREEN — read-only device capabilities"
  - "create-quantum-task (resubmit): YELLOW — incurs per-minute simulation charges"
  - "local simulator testing: GREEN — no cloud charges"
```

## Escalation Conditions

- Simulator returns internal errors on circuits well within documented limits
- Same circuit fails on all three simulators with different errors
- Simulator performance degrades significantly compared to previous runs
- Error messages reference internal service components

## Data Sensitivity

- `aws braket get-quantum-task`: LOW — returns task metadata
- `aws braket get-device`: LOW — returns simulator capabilities
- Circuit definitions: LOW — quantum circuit structure

## Prohibited Actions

- NEVER submit large circuits to simulators without estimating cost (per-minute billing)
- NEVER assume TN1 can simulate any 50-qubit circuit — structure matters
- NEVER use DM1 for circuits exceeding 17 qubits — it will fail
- NEVER ignore simulator-specific constraints when migrating circuits between simulators

## Phase 3 — Rollback

1. If a costly simulation was started in error, cancel if still queued: `aws braket cancel-quantum-task --quantum-task-arn <arn>`
2. Revert circuit modifications made during troubleshooting if they changed the algorithm
3. Document simulator-specific constraints encountered for the circuit
4. Verify billing for any simulation time consumed during troubleshooting
5. Restore original circuit definition if it was modified for debugging

## Common Issues

- symptoms: "TN1 task fails or runs extremely long on a highly entangled circuit"
  diagnosis: "TN1 tensor network contraction is inefficient for highly entangled circuits."
  resolution: "Use SV1 for highly entangled circuits under 34 qubits. TN1 works best for sparse connectivity."

- symptoms: "DM1 fails with qubit limit error"
  diagnosis: "DM1 density matrix simulator supports maximum 17 qubits."
  resolution: "Reduce circuit to 17 qubits or switch to SV1 (up to 34 qubits) if noise simulation is not required."

## Output Format

```yaml
root_cause: "simulator_error — <specific_cause>"
evidence:
  - type: simulator_failure
    content: "<failureReason and simulator type>"
severity: MEDIUM
mitigation:
  immediate: "Fix circuit or switch simulator"
  long_term: "Implement simulator selection logic based on circuit properties"
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

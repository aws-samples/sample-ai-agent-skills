# AWS IoT Events Diagnostics Skill

Agent skill for investigating and troubleshooting AWS IoT Events problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS IoT Events — detector models, inputs, alarms, actions, state transitions, Lambda actions, and event evaluation.

### Activate When

- Detector model creation or validation failures
- Input not reaching detector model
- Alarm not evaluating or stuck in wrong state
- Action execution failures (SNS, SQS, Lambda)
- State transitions not firing
- Condition expression errors
- Lambda action timeouts or errors
- General IoT Events errors

---

## Skill Structure

```
iotevents-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-detector-model-creation.md
    ├── A2-model-validation.md
    ├── B1-input-routing.md
    ├── B2-input-schema.md
    ├── C1-alarm-evaluation.md
    ├── C2-alarm-state-issues.md
    ├── D1-action-execution.md
    ├── D2-lambda-actions.md
    ├── E1-transition-logic.md
    ├── E2-condition-errors.md
    ├── Z1-general-troubleshooting.md
    ├── Z2-performance-issues.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Detector Models** | A1–A2 | Model creation, validation errors |
| **B — Inputs** | B1–B2 | Input routing, schema issues |
| **C — Alarms** | C1–C2 | Alarm evaluation, state issues |
| **D — Actions** | D1–D2 | Action execution, Lambda actions |
| **E — State Transitions** | E1–E2 | Transition logic, condition errors |
| **Z — Catch-All** | Z1–Z2 | General troubleshooting, performance |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering detector model state machine semantics, input schema matching, detector instance isolation, action IAM role requirements, alarm state machine behavior, condition expression syntax, timer behavior, variable scope, payload size limits, evaluation order, action ordering, and detector deletion behavior.

---

## Prerequisites

- AWS CLI v2 with IoTEvents permissions
- Permissions: `iotevents:Describe*`, `iotevents:List*`, `iotevents-data:Describe*`, `iotevents-data:List*`

---

## License

MIT-0

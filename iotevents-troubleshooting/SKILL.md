---
name: iotevents-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS IoT Events problems
  by analyzing detector models, inputs, alarms, actions, state transitions,
  Lambda actions, and event evaluation using structured runbooks. Activate
  when: detector model creation failures, input routing errors, alarm
  evaluation issues, action execution failures, state transition problems,
  or the user says something is wrong with IoT Events.
compatibility: >
  Requires AWS CLI or SDK access with IoTEvents, IAM, CloudWatch,
  Lambda, SNS, and SQS permissions as applicable.
---

# AWS IoT Events Diagnostics

## When to use

Any AWS IoT Events investigation — detector models, inputs, alarms, actions, state transitions, Lambda actions, or event evaluation troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws iotevents list-detector-models
aws iotevents describe-detector-model --detector-model-name <name>
aws iotevents list-inputs
aws iotevents describe-input --input-name <name>
```

### Step 2 — Domain deep dive

```
aws iotevents-data list-detectors --detector-model-name <name>
aws iotevents-data describe-detector --detector-model-name <name> --key-value <key>
aws iotevents list-alarm-models
aws iotevents describe-alarm-model --alarm-model-name <name>
```

### Step 3 — Detailed investigation

```
aws iotevents-data list-alarms --alarm-model-name <name>
aws iotevents-data describe-alarm --alarm-model-name <name> --key-value <key>
aws cloudwatch get-metric-statistics --namespace AWS/IoTEvents --metric-name ActionExecutionError --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=iotevents.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any IoT Events issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-detector-model` | Check model definition and states |
| `describe-input` | Check input schema |
| `describe-detector` | Check runtime detector state |
| `describe-alarm-model` | Check alarm configuration |
| `describe-alarm` | Check alarm runtime state |
| CloudWatch Metrics | Check action errors and evaluation |

## Gotchas: AWS IoT Events

- Detector models are state machines. Each state has onEnter, onInput, and onExit events. Transitions are evaluated on each input message.
- Inputs define the schema for incoming data. Input attribute paths must match the JSON payload structure exactly. Case-sensitive matching.
- Detectors are instances of detector models. Each unique key value creates a separate detector instance with its own state.
- Actions execute on events (onEnter, onInput, onExit) and transitions. Actions include SNS, SQS, Lambda, IoT topic publish, and more.
- Alarm models are simplified detector models for threshold monitoring. They have predefined states (NORMAL, ACTIVE, ACKNOWLEDGED, etc.).
- Conditions use a specific expression language, not standard SQL. Expressions support comparison, logical, and arithmetic operators.

## Anti-hallucination rules

1. Always cite specific detector model names, input names, or API responses as evidence.
2. Detector model conditions use IoT Events expression language, not SQL.
3. Each key value creates a separate detector instance. Never assume shared state.
4. Actions require IAM role permissions. Never assume automatic access.
5. Alarm models have predefined state machines. Never suggest custom alarm states.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Detector Models | A1-A2 | Model creation, validation errors |
| B — Inputs | B1-B2 | Input routing, schema issues |
| C — Alarms | C1-C2 | Alarm evaluation, state issues |
| D — Actions | D1-D2 | Action execution, Lambda actions |
| E — State Transitions | E1-E2 | Transition logic, condition errors |
| Z — Catch-All | Z1-Z2 | General troubleshooting, performance |

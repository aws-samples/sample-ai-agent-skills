---
name: braket-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Braket
  quantum computing problems by analyzing quantum task failures,
  simulator errors, hardware device issues, circuit validation,
  hybrid jobs, notebook instances, SDK errors, cost management,
  device availability, and result retrieval using structured runbooks.
  Activate when: quantum task failures, task timeouts, simulator
  errors, hardware device problems, circuit validation failures,
  hybrid job issues, notebook instance problems, SDK errors,
  cost concerns, device availability questions, result retrieval
  failures, or the user says something is wrong with Braket.
compatibility: >
  Requires AWS CLI or SDK access with braket, iam, s3, logs,
  cloudwatch, and sagemaker permissions.
---

# Amazon Braket Diagnostics

## When to use

Any Amazon Braket investigation — quantum task failures, task timeouts, simulator errors, hardware device issues, circuit validation, hybrid job failures, notebook instance problems, SDK errors, cost management, device availability, or result retrieval.

## Investigation workflow

### Step 1 — Collect and triage

```
aws braket search-quantum-tasks --filters '[{"name":"status","operator":"EQUAL","values":["FAILED"]}]'
aws braket search-devices --filters '[{"name":"deviceType","operator":"EQUAL","values":["QPU"]}]'
aws braket get-quantum-task --quantum-task-arn <arn>
```

### Step 2 — Domain deep dive

```
aws braket get-device --device-arn <arn>
aws braket search-jobs --filters '[{"name":"status","operator":"EQUAL","values":["FAILED"]}]'
aws s3 ls s3://<braket-output-bucket>/tasks/<task-id>/
```

### Step 3 — Detailed investigation

```
aws cloudwatch get-metric-statistics --namespace "AWS/Braket" --metric-name "TasksCompleted" --dimensions Name=DeviceArn,Value=<arn> --start-time <start> --end-time <end> --period 3600 --statistics Sum
aws logs filter-log-events --log-group-name /aws/braket/jobs/<job-name> --limit 50
aws braket get-job --job-arn <arn>
```

Read `references/guardrails.md` before concluding on any Braket issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `braket search-quantum-tasks` | List quantum tasks with filters |
| `braket get-quantum-task` | Get task details and results |
| `braket search-devices` | List available devices |
| `braket get-device` | Get device capabilities and status |
| `braket search-jobs` | List hybrid jobs |
| `braket get-job` | Get hybrid job details |
| `braket cancel-quantum-task` | Cancel a running task |

## Gotchas: Amazon Braket

- Quantum tasks on QPU devices are queued and may take hours depending on device availability windows. Simulators are on-demand.
- Each QPU device has specific availability windows. Tasks submitted outside windows queue until the next window opens.
- Braket results are stored in S3. If the output bucket is misconfigured or permissions are wrong, tasks succeed but results are inaccessible.
- Simulator costs are per-minute of simulation time. QPU costs are per-task plus per-shot. Costs vary dramatically between devices.
- Hybrid jobs run on classical compute (ml.m5.xlarge default) plus quantum device access. Both incur separate charges.
- Circuit depth and qubit count limits vary by device. A circuit valid on one device may fail on another.
- The Braket SDK (amazon-braket-sdk) version must be compatible with the service. Outdated SDKs may produce unexpected errors.

## Anti-hallucination rules

1. Always cite specific quantum task ARNs, device ARNs, or API responses as evidence.
2. QPU and simulator devices have fundamentally different behavior, pricing, and availability. Never conflate them.
3. Device availability windows change frequently. Never state specific availability times without checking.
4. Braket does NOT provide error correction. Never claim quantum error correction is built in.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Quantum Tasks | A1–A2 | Task failures, task timeouts |
| B — Devices | B1–B2 | Simulator errors, hardware device issues |
| C — Circuits | C1–C2 | Circuit validation, hybrid job failures |
| D — Environment | D1–D2 | Notebook instance issues, SDK errors |
| E — Operations | E1–E2 | Cost management, device availability |
| F — Results | F1 | Result retrieval |
| Z — Catch-All | Z1 | General troubleshooting |

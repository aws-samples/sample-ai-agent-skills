# AWS IoT Device Management Diagnostics Skill

Agent skill for investigating and troubleshooting AWS IoT Device Management problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS IoT Device Management — fleet provisioning, bulk registration, job execution, secure tunneling, thing groups, thing types, fleet indexing, device shadows, and OTA updates.

### Activate When

- Fleet provisioning failures
- Bulk registration errors
- Job execution failures or timeouts
- Secure tunneling connection issues
- Thing group or thing type problems
- Fleet indexing configuration errors
- Device shadow sync failures
- OTA update failures
- General IoT Device Management errors

---

## Skill Structure

```
iot-device-management-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-fleet-provisioning-failures.md
    ├── A2-bulk-registration-errors.md
    ├── B1-job-execution-failures.md
    ├── B2-job-timeout-issues.md
    ├── C1-secure-tunneling-errors.md
    ├── C2-tunnel-connectivity.md
    ├── D1-thing-group-issues.md
    ├── D2-thing-type-configuration.md
    ├── E1-fleet-indexing-errors.md
    ├── E2-device-shadow-sync.md
    ├── F1-ota-update-failures.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Fleet Provisioning** | A1–A2 | Provisioning failures, bulk registration |
| **B — Jobs** | B1–B2 | Execution failures, timeouts |
| **C — Secure Tunneling** | C1–C2 | Tunneling errors, connectivity |
| **D — Thing Management** | D1–D2 | Thing groups, thing types |
| **E — Fleet Indexing & Shadows** | E1–E2 | Indexing errors, shadow sync |
| **F — OTA Updates** | F1 | OTA update failures |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `iot:*`, `iotsecuretunneling:*`, `iot-data:*`, `s3:Get*`, `iam:Get*`, `cloudwatch:GetMetricData` permissions

---

## License

MIT-0

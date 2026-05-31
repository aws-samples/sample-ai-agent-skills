# AWS IoT Greengrass V2 Diagnostics Skill

Agent skill for investigating and troubleshooting AWS IoT Greengrass V2 problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS IoT Greengrass V2 — core device provisioning, component deployments, Lambda components, MQTT bridge, stream manager, local shadow service, OTA updates, nucleus configuration, and edge runtime issues.

### Activate When

- Core device provisioning or registration failures
- Greengrass nucleus startup errors
- Component deployment failures or conflicts
- Lambda component crashes or timeout
- MQTT bridge not forwarding messages
- Stream manager export failures
- Local shadow sync issues
- OTA update failures or rollback
- Token exchange service credential errors
- IPC communication failures between components
- General Greengrass errors without clear symptoms

---

## Skill Structure

```
iotgreengrass-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-core-device-provisioning.md
    ├── A2-nucleus-errors.md
    ├── A3-device-connectivity.md
    ├── B1-deployment-failures.md
    ├── B2-component-conflicts.md
    ├── B3-rollback-issues.md
    ├── C1-lambda-component-errors.md
    ├── C2-lambda-runtime-issues.md
    ├── D1-mqtt-bridge-config.md
    ├── D2-topic-mapping.md
    ├── E1-stream-export-failures.md
    ├── E2-stream-storage.md
    ├── F1-local-shadow-sync.md
    ├── F2-cloud-shadow-sync.md
    ├── G1-ota-update-failures.md
    ├── G2-ota-rollback.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (16 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Core Device** | A1–A3 | Provisioning failures, nucleus errors, connectivity |
| **B — Deployment** | B1–B3 | Deployment failures, component conflicts, rollback |
| **C — Lambda** | C1–C2 | Lambda component errors, runtime issues |
| **D — MQTT Bridge** | D1–D2 | Bridge configuration, topic mapping |
| **E — Stream Manager** | E1–E2 | Export failures, storage issues |
| **F — Shadow** | F1–F2 | Local shadow sync, cloud shadow sync |
| **G — OTA Updates** | G1–G2 | Update failures, rollback issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering V2 vs V1 differences, nucleus requirements, component recipe validation, token exchange role permissions, MQTT bridge topic mapping, stream manager limits, local shadow IPC, OTA deployment targeting, component dependency resolution, artifact storage, device certificate requirements, and rollback configuration.

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `greengrass:*`, `iot:Describe*`, `iot:List*`, `iam:GetRole*`, `logs:FilterLogEvents`, `cloudwatch:GetMetricStatistics`
- Access to core device logs at `/greengrass/v2/logs/`

---

## License

MIT-0

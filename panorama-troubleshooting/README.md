# AWS Panorama Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Panorama problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Panorama — device provisioning, application deployment, model compilation, camera stream management, OTA updates, and network connectivity.

### Activate When

- Device provisioning or registration failures
- Application deployment errors or timeouts
- Model compilation failures
- Camera stream disconnections or configuration issues
- OTA update failures or stuck updates
- Network connectivity problems
- Device showing offline status
- General Panorama errors without clear symptoms

---

## Skill Structure

```
panorama-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-device-provisioning.md
    ├── A2-registration-errors.md
    ├── B1-app-deployment.md
    ├── B2-app-lifecycle.md
    ├── C1-model-compilation.md
    ├── C2-model-import.md
    ├── D1-stream-configuration.md
    ├── D2-stream-failures.md
    ├── E1-ota-updates.md
    ├── E2-firmware-issues.md
    ├── F1-connectivity.md
    ├── F2-dns-proxy.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `panorama:*`, `s3:GetObject`, `iam:PassRole`, `logs:FilterLogEvents`, `cloudwatch:GetMetricStatistics`
- Physical Panorama appliance for device operations

---

## License

MIT-0

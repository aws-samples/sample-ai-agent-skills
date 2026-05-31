# AWS Elemental Link Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Elemental Link hardware encoder problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Elemental Link — device registration, connectivity, input configuration, video quality, network settings, HDMI/SDI inputs, MediaLive integration, firmware updates, monitoring, and multi-device management.

### Activate When

- Device registration failures
- Device connectivity issues
- Input configuration errors
- Video quality degradation
- Network configuration problems
- HDMI/SDI input errors
- MediaLive integration failures
- Channel attachment issues
- Firmware update problems
- Device monitoring gaps
- Multi-device management issues
- General Elemental Link errors

---

## Skill Structure

```
elemental-link-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-device-registration-failures.md
    ├── A2-device-connectivity.md
    ├── B1-input-configuration-errors.md
    ├── B2-video-quality-issues.md
    ├── C1-network-configuration.md
    ├── C2-hdmi-sdi-input-errors.md
    ├── D1-medialive-integration.md
    ├── D2-channel-attachment-issues.md
    ├── E1-device-firmware-updates.md
    ├── E2-device-monitoring.md
    ├── F1-multi-device-management.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Registration** | A1–A2 | Registration failures, connectivity |
| **B — Input** | B1–B2 | Configuration errors, video quality |
| **C — Network** | C1–C2 | Network config, HDMI/SDI errors |
| **D — Integration** | D1–D2 | MediaLive integration, channel attachment |
| **E — Maintenance** | E1–E2 | Firmware updates, monitoring |
| **F — Fleet** | F1 | Multi-device management |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `medialive:*`, `cloudwatch:GetMetricStatistics`, `ec2:Describe*`, `iam:Get*` permissions
- Physical access to Elemental Link hardware for HDMI/SDI and network troubleshooting

---

## License

MIT-0

# AWS Private 5G Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Private 5G problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Private 5G — network creation, network sites, radio unit activation, device connectivity, SIM provisioning, subscriber management, throughput, coverage, access point configuration, network health, and VPC integration.

### Activate When

- Network creation failures
- Network site issues
- Radio unit activation problems
- Device connectivity failures
- SIM provisioning errors
- Subscriber management issues
- Throughput degradation
- Coverage problems
- Access point misconfiguration
- Network health alerts
- VPC integration failures
- General Private 5G errors

---

## Skill Structure

```
private5g-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-network-creation-failures.md
    ├── A2-network-site-issues.md
    ├── B1-radio-unit-activation.md
    ├── B2-device-connectivity.md
    ├── C1-sim-provisioning.md
    ├── C2-subscriber-management.md
    ├── D1-throughput-issues.md
    ├── D2-coverage-problems.md
    ├── E1-access-point-configuration.md
    ├── E2-network-health.md
    ├── F1-integration-with-vpc.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Network** | A1–A2 | Creation failures, site issues |
| **B — Radio & Devices** | B1–B2 | Radio unit activation, device connectivity |
| **C — SIM & Subscribers** | C1–C2 | SIM provisioning, subscriber management |
| **D — Performance** | D1–D2 | Throughput issues, coverage problems |
| **E — Configuration** | E1–E2 | Access point configuration, network health |
| **F — Integration** | F1 | VPC integration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `private-networks:*`, `ec2:Describe*`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents` permissions

---

## License

MIT-0

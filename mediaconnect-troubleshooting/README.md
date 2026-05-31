# AWS Elemental MediaConnect Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Elemental MediaConnect problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Elemental MediaConnect — reliable live video transport including flow management, source connectivity, output delivery, entitlements, VPC interfaces, encryption, transport streams, bandwidth, monitoring, maintenance events, and multi-region flows.

### Activate When

- Flow creation failures
- Flow configuration issues
- Source connectivity problems
- Source failover not working
- Output delivery failures
- Entitlement sharing issues
- VPC interface errors
- Encryption misconfigurations
- Transport stream errors
- Bandwidth constraints
- CloudWatch monitoring gaps
- Maintenance event handling
- Multi-region flow issues
- General MediaConnect errors

---

## Skill Structure

```
mediaconnect-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-flow-creation-failures.md
    ├── A2-flow-configuration.md
    ├── B1-source-connectivity.md
    ├── B2-source-failover.md
    ├── C1-output-failures.md
    ├── C2-entitlement-issues.md
    ├── D1-vpc-interface-errors.md
    ├── D2-encryption-issues.md
    ├── E1-transport-stream-errors.md
    ├── E2-bandwidth-issues.md
    ├── F1-cloudwatch-monitoring.md
    ├── F2-maintenance-events.md
    ├── G1-multi-region-flows.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Flow Management** | A1–A2 | Creation failures, configuration |
| **B — Source** | B1–B2 | Connectivity, failover |
| **C — Output** | C1–C2 | Output failures, entitlements |
| **D — Network & Security** | D1–D2 | VPC interfaces, encryption |
| **E — Transport & Performance** | E1–E2 | Transport stream errors, bandwidth |
| **F — Operations** | F1–F2 | CloudWatch monitoring, maintenance events |
| **G — Multi-Region** | G1 | Multi-region flows |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `mediaconnect:*`, `ec2:Describe*`, `kms:Describe*`, `cloudwatch:GetMetricStatistics`, `iam:Get*`, `logs:*` permissions

---

## License

MIT-0

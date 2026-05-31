# CloudWatch Application Signals Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon CloudWatch Application Signals problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for CloudWatch Application Signals — service discovery, instrumentation, SLO monitoring, service dependency mapping, metric collection, trace and log correlation, EKS/EC2 integration, and alarm configuration.

### Activate When

- Service discovery failures
- Instrumentation errors
- SLO creation issues
- SLO evaluation errors
- Service dependency mapping problems
- Metric collection failures
- Trace correlation issues
- Log correlation errors
- EKS integration problems
- EC2 integration issues
- Alarm configuration failures
- General Application Signals errors

---

## Skill Structure

```
cloudwatch-application-signals-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-service-discovery-failures.md
    ├── A2-instrumentation-errors.md
    ├── B1-slo-creation-issues.md
    ├── B2-slo-evaluation-errors.md
    ├── C1-service-dependency-mapping.md
    ├── C2-metric-collection-failures.md
    ├── D1-trace-correlation-issues.md
    ├── D2-log-correlation-errors.md
    ├── E1-eks-integration.md
    ├── E2-ec2-integration.md
    ├── F1-alarm-configuration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Discovery & Instrumentation** | A1–A2 | Service discovery, instrumentation errors |
| **B — SLOs** | B1–B2 | SLO creation, SLO evaluation |
| **C — Dependencies & Metrics** | C1–C2 | Dependency mapping, metric collection |
| **D — Correlation** | D1–D2 | Trace correlation, log correlation |
| **E — Integration** | E1–E2 | EKS integration, EC2 integration |
| **F — Alarms** | F1 | Alarm configuration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `application-signals:*`, `cloudwatch:*`, `xray:*`, `logs:*`, `eks:Describe*`, `ec2:Describe*`, `iam:Get*` permissions

---

## License

MIT-0

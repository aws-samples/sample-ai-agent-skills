# Elastic Fabric Adapter (EFA) Diagnostics Skill

Agent skill for investigating and troubleshooting EFA problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Elastic Fabric Adapter — high-performance networking for HPC and ML workloads. Covers EFA creation, attachment, drivers, libfabric, MPI, NCCL, performance, bandwidth, security groups, placement groups, and SageMaker integration.

### Activate When

- EFA creation or attachment failures
- Driver installation issues
- Libfabric provider errors
- MPI communication failures
- NCCL collective operation errors
- Performance degradation
- Bandwidth limitations
- Security group misconfigurations
- Placement group issues
- SageMaker EFA integration problems
- General EFA errors

---

## Skill Structure

```
efa-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-efa-creation-failures.md
    ├── A2-efa-attachment-errors.md
    ├── B1-driver-installation.md
    ├── B2-libfabric-issues.md
    ├── C1-mpi-communication-errors.md
    ├── C2-nccl-errors.md
    ├── D1-performance-issues.md
    ├── D2-bandwidth-limitations.md
    ├── E1-security-group-configuration.md
    ├── E2-placement-group-issues.md
    ├── F1-sagemaker-integration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Creation & Attachment** | A1–A2 | Creation failures, attachment errors |
| **B — Drivers & Libraries** | B1–B2 | Driver installation, libfabric |
| **C — Communication** | C1–C2 | MPI errors, NCCL errors |
| **D — Performance** | D1–D2 | Performance, bandwidth |
| **E — Network Config** | E1–E2 | Security groups, placement groups |
| **F — Integration** | F1 | SageMaker integration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `ec2:Describe*`, `ec2:CreateNetworkInterface`, `sagemaker:Describe*` permissions
- SSH or SSM access to instances for driver-level diagnostics
- EFA installer package for driver and libfabric installation

---

## License

MIT-0

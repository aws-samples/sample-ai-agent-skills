# AWS Backup Gateway Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Backup Gateway problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Backup Gateway — gateway deployment, connectivity, hypervisor connections, vCenter authentication, VM discovery, VM backups, restores, performance, bandwidth, network configuration, and gateway maintenance.

### Activate When

- Gateway deployment failures
- Gateway connectivity issues
- Hypervisor connection errors
- vCenter authentication failures
- VM discovery problems
- VM backup errors
- Restore failures
- Restore performance issues
- Bandwidth throttling
- Network configuration problems
- Gateway maintenance issues
- General Backup Gateway errors

---

## Skill Structure

```
backup-gateway-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-gateway-deployment-failures.md
    ├── A2-gateway-connectivity.md
    ├── B1-hypervisor-connection-errors.md
    ├── B2-vcenter-authentication.md
    ├── C1-vm-discovery-failures.md
    ├── C2-vm-backup-errors.md
    ├── D1-restore-failures.md
    ├── D2-restore-performance.md
    ├── E1-bandwidth-throttling.md
    ├── E2-network-configuration.md
    ├── F1-gateway-maintenance.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Gateway** | A1–A2 | Deployment, connectivity |
| **B — Hypervisor** | B1–B2 | Connection, vCenter auth |
| **C — VMs** | C1–C2 | Discovery, backup errors |
| **D — Restore** | D1–D2 | Failures, performance |
| **E — Network** | E1–E2 | Bandwidth, network config |
| **F — Maintenance** | F1 | Gateway maintenance |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `backup-gateway:*`, `backup:Get*`, `ec2:Describe*`, `iam:Get*`, `cloudwatch:GetMetricStatistics` permissions
- Network access to on-premises VMware environment for gateway troubleshooting

---

## License

MIT-0

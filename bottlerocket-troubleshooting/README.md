# Bottlerocket Diagnostics Skill

Agent skill for investigating and troubleshooting Bottlerocket OS problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Bottlerocket — the purpose-built Linux OS for running containers on ECS and EKS. Covers AMI launches, OS updates, container agents, admin/control container access, networking, storage, settings API, bootstrap, and security hardening.

### Activate When

- AMI launch failures
- OS update failures or rollback issues
- ECS agent connectivity problems
- Kubelet or EKS node issues
- Admin container access problems
- Control container or SSM issues
- Networking misconfigurations
- Storage or volume mount errors
- Settings API errors
- Bootstrap failures
- Security hardening questions
- General Bottlerocket errors

---

## Skill Structure

```
bottlerocket-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-ami-launch-failures.md
    ├── A2-update-failures.md
    ├── B1-ecs-agent-issues.md
    ├── B2-kubelet-issues.md
    ├── C1-admin-container-access.md
    ├── C2-control-container-issues.md
    ├── D1-networking-configuration.md
    ├── D2-storage-issues.md
    ├── E1-settings-api-errors.md
    ├── E2-bootstrap-failures.md
    ├── F1-security-hardening.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — AMI & Updates** | A1–A2 | Launch failures, update failures |
| **B — Container Agents** | B1–B2 | ECS agent, kubelet |
| **C — Access** | C1–C2 | Admin container, control container |
| **D — Infrastructure** | D1–D2 | Networking, storage |
| **E — Configuration** | E1–E2 | Settings API, bootstrap |
| **F — Security** | F1 | Security hardening |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `ec2:Describe*`, `ssm:StartSession`, `ecs:Describe*`, `eks:Describe*`, `cloudwatch:GetMetricStatistics` permissions
- SSM Session Manager plugin installed for host-level diagnostics

---

## License

MIT-0

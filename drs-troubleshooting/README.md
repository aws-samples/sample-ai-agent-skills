# AWS Elastic Disaster Recovery Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Elastic Disaster Recovery (DRS) problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS DRS when the console alone isn't enough — source server registration and agent issues, replication lag and stalls, launch configuration errors, recovery instance failures, failback operations, drill testing, and networking connectivity.

### Activate When

- Source server not appearing in DRS console
- Replication agent installation failures
- Replication lag or stalled replication
- Initial sync not completing
- Recovery launch failures
- Recovery instance not booting correctly
- Failback initiation or replication problems
- Drill test launch failures
- Drill instance cleanup issues
- Staging area networking problems
- General DRS errors without clear symptoms

---

## Skill Structure

```
drs-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-source-server-registration.md
    ├── A2-agent-installation.md
    ├── A3-source-properties.md
    ├── B1-replication-lag.md
    ├── B2-stalled-replication.md
    ├── B3-initial-sync.md
    ├── C1-launch-template-errors.md
    ├── C2-instance-type-mapping.md
    ├── D1-recovery-launch-failures.md
    ├── D2-recovery-instance-issues.md
    ├── D3-post-launch-actions.md
    ├── E1-failback-initiation.md
    ├── E2-failback-replication.md
    ├── F1-drill-launch-failures.md
    ├── F2-drill-cleanup.md
    ├── G1-networking-connectivity.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (16 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Source Server** | A1–A3 | Registration, agent issues, source properties |
| **B — Replication** | B1–B3 | Replication lag, stalled replication, initial sync |
| **C — Launch Config** | C1–C2 | Launch template errors, instance type mapping |
| **D — Recovery** | D1–D3 | Recovery launch failures, instance issues, post-launch |
| **E — Failback** | E1–E2 | Failback initiation, failback replication |
| **F — Drill Testing** | F1–F2 | Drill launch failures, drill cleanup |
| **G — Networking** | G1 | Staging area and connectivity |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering agent installation prerequisites, replication server management, initial sync expectations, staging area networking, launch configuration scope, drill instance lifecycle, failback client requirements, recovery instance boot validation, replication bandwidth, source server lifecycle states, data replication lag interpretation, and cost implications.

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `drs:*`, `ec2:Describe*`, `iam:Get*`, `cloudtrail:LookupEvents`
- Source servers with AWS Replication Agent installed
- Staging area subnet with required connectivity

---

## License

MIT-0

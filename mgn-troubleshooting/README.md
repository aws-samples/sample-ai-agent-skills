# AWS Application Migration Service Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Application Migration Service (MGN) problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS MGN — source server registration, replication lag and stalls, launch template errors, cutover operations, post-migration validation, and agent installation issues.

### Activate When

- Source server not appearing in MGN console
- Replication lag or stalled replication
- Launch template configuration errors
- Cutover launch failures
- Post-migration application issues
- Agent installation or connectivity problems
- General MGN errors without clear symptoms

---

## Skill Structure

```
mgn-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-source-server-registration.md
    ├── A2-source-properties.md
    ├── B1-replication-lag.md
    ├── B2-stalled-replication.md
    ├── B3-initial-sync.md
    ├── C1-launch-template-errors.md
    ├── C2-instance-type-mapping.md
    ├── D1-cutover-launch.md
    ├── D2-cutover-finalization.md
    ├── D3-cutover-rollback.md
    ├── E1-post-migration-validation.md
    ├── E2-application-testing.md
    ├── F1-agent-installation.md
    ├── F2-agent-connectivity.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Source Server** | A1–A2 | Registration, source properties |
| **B — Replication** | B1–B3 | Lag, stalled, initial sync |
| **C — Launch Template** | C1–C2 | Launch errors, instance mapping |
| **D — Cutover** | D1–D3 | Launch, finalization, rollback |
| **E — Post-Migration** | E1–E2 | Validation, application testing |
| **F — Agent** | F1–F2 | Installation, connectivity |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `mgn:*`, `ec2:Describe*`, `ssm:*`, `iam:Get*` permissions
- Source servers with AWS Replication Agent installed

---

## License

MIT-0

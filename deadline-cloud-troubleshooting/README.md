# AWS Deadline Cloud Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Deadline Cloud problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Deadline Cloud — farm creation, fleet configuration, queue management, job submission, worker agents, storage profiles, job attachments, budgets, license endpoints, monitors, identity management, and custom job templates.

### Activate When

- Farm creation failures
- Fleet configuration issues
- Queue management problems
- Job submission errors
- Worker agent failures
- Worker host issues
- Storage profile errors
- Job attachment problems
- Budget configuration issues
- License endpoint errors
- Monitor configuration failures
- Identity management problems
- Custom job template errors
- General Deadline Cloud errors

---

## Skill Structure

```
deadline-cloud-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-farm-creation-failures.md
    ├── A2-fleet-configuration.md
    ├── B1-queue-issues.md
    ├── B2-job-submission-errors.md
    ├── C1-worker-agent-failures.md
    ├── C2-worker-host-issues.md
    ├── D1-storage-profile-errors.md
    ├── D2-job-attachment-issues.md
    ├── E1-budget-configuration.md
    ├── E2-license-endpoint-errors.md
    ├── F1-monitor-configuration.md
    ├── F2-identity-management.md
    ├── G1-custom-job-templates.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Farm & Fleet** | A1–A2 | Farm creation, fleet configuration |
| **B — Queue & Jobs** | B1–B2 | Queue issues, job submission errors |
| **C — Workers** | C1–C2 | Worker agent failures, worker host issues |
| **D — Storage** | D1–D2 | Storage profiles, job attachments |
| **E — Budget & License** | E1–E2 | Budget configuration, license endpoints |
| **F — Monitor & Identity** | F1–F2 | Monitor configuration, identity management |
| **G — Templates** | G1 | Custom job templates |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `deadline:*`, `ec2:Describe*`, `s3:Get*`, `iam:Get*`, `logs:FilterLogEvents`, `cloudwatch:GetMetricStatistics` permissions

---

## License

MIT-0

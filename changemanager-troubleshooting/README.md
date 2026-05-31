# AWS SSM Change Manager Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Systems Manager Change Manager problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for SSM Change Manager — change templates, change requests, approval workflows, runbook execution, and scheduling.

### Activate When

- Change template creation or approval failures
- Change request submission errors
- Approval workflow not progressing
- Approvers not receiving notifications
- Runbook execution failures during changes
- Scheduled changes not executing
- General Change Manager errors

---

## Skill Structure

```
changemanager-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-template-creation.md
    ├── A2-template-approval.md
    ├── B1-request-submission.md
    ├── B2-request-status.md
    ├── C1-approval-workflow.md
    ├── C2-approver-notification.md
    ├── C3-approval-timeout.md
    ├── D1-runbook-execution.md
    ├── D2-execution-failures.md
    ├── E1-schedule-configuration.md
    ├── E2-schedule-conflicts.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Templates** | A1–A2 | Creation, approval |
| **B — Requests** | B1–B2 | Submission, status |
| **C — Approvals** | C1–C3 | Workflow, notification, timeout |
| **D — Execution** | D1–D2 | Runbook execution, failures |
| **E — Scheduling** | E1–E2 | Configuration, conflicts |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `ssm:*`, `iam:Get*`, `sns:*`, `cloudtrail:LookupEvents` permissions

---

## License

MIT-0

# AWS Budgets Actions Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Budgets Actions problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Budgets Actions — automated cost control actions triggered by budget thresholds including IAM policy actions, SCP actions, EC2/RDS instance actions, approval workflows, and notification configuration.

### Activate When

- Budget creation failures
- Budget alert configuration issues
- Action execution failures
- IAM policy action errors
- SCP action failures
- EC2 instance action errors
- RDS instance action errors
- Approval workflow issues
- Threshold configuration problems
- Notification failures
- Organization budget issues
- General Budgets Actions errors

---

## Skill Structure

```
budgets-actions-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-budget-creation-failures.md
    ├── A2-budget-alert-configuration.md
    ├── B1-action-execution-failures.md
    ├── B2-iam-policy-action-errors.md
    ├── C1-scp-action-failures.md
    ├── C2-ec2-instance-action-errors.md
    ├── D1-rds-instance-action-errors.md
    ├── D2-approval-workflow-issues.md
    ├── E1-threshold-configuration.md
    ├── E2-notification-failures.md
    ├── F1-organization-budgets.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Budget Setup** | A1–A2 | Budget creation, alert configuration |
| **B — Action Execution** | B1–B2 | Execution failures, IAM policy actions |
| **C — SCP & EC2 Actions** | C1–C2 | SCP actions, EC2 instance actions |
| **D — RDS & Approvals** | D1–D2 | RDS actions, approval workflows |
| **E — Thresholds & Notifications** | E1–E2 | Thresholds, notifications |
| **F — Organization** | F1 | Organization budgets |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `budgets:*`, `iam:Get*`, `ec2:Describe*`, `rds:Describe*`, `sns:*`, `organizations:Describe*` permissions

---

## License

MIT-0

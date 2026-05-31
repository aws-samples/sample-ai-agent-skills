# SSM Automation Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Systems Manager Automation problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for SSM Automation — execution failures, step errors, document creation, versioning, approval workflows, rate control, cross-account automation, multi-region execution, change calendar integration, maintenance window automation, custom actions, output parameters, and service roles.

### Activate When

- Automation execution failures
- Step execution errors
- Document creation issues
- Document versioning problems
- Approval workflow failures
- Rate control issues
- Cross-account automation errors
- Multi-region execution failures
- Change calendar blocking executions
- Maintenance window automation issues
- Custom action errors
- Output parameter issues
- Service role problems
- General SSM Automation errors

---

## Skill Structure

```
ssm-automation-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-automation-execution-failures.md
    ├── A2-step-execution-errors.md
    ├── B1-document-creation-issues.md
    ├── B2-document-versioning.md
    ├── C1-approval-workflow-failures.md
    ├── C2-rate-control-issues.md
    ├── D1-cross-account-automation.md
    ├── D2-multi-region-execution.md
    ├── E1-change-calendar-integration.md
    ├── E2-maintenance-window-automation.md
    ├── F1-custom-action-errors.md
    ├── F2-output-parameter-issues.md
    ├── G1-automation-service-role.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Execution** | A1–A2 | Execution failures, step errors |
| **B — Documents** | B1–B2 | Creation issues, versioning |
| **C — Workflows** | C1–C2 | Approval failures, rate control |
| **D — Multi-Scope** | D1–D2 | Cross-account, multi-region |
| **E — Scheduling** | E1–E2 | Change calendar, maintenance windows |
| **F — Advanced** | F1–F2 | Custom actions, output parameters |
| **G — IAM** | G1 | Automation service role |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `ssm:*`, `iam:Get*`, `sts:AssumeRole`, `organizations:Describe*`, `cloudtrail:LookupEvents`, `cloudwatch:GetMetricData` permissions

---

## License

MIT-0

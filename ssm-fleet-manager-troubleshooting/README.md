# AWS Systems Manager Fleet Manager Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Systems Manager Fleet Manager problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Fleet Manager — managed node registration, node visibility, remote desktop, file system access, performance counters, OS patching visibility, hybrid activations, IAM instance profiles, SSM Agent connectivity, and account-level settings.

### Activate When

- Managed nodes not appearing in Fleet Manager
- Remote desktop connection failures
- File system access errors
- Performance counter issues
- OS patching visibility problems
- Hybrid activation errors
- IAM instance profile misconfiguration
- SSM Agent connectivity problems
- Account-level settings issues
- General Fleet Manager errors

---

## Skill Structure

```
ssm-fleet-manager-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-managed-node-registration.md
    ├── A2-node-not-showing.md
    ├── B1-remote-desktop-failures.md
    ├── B2-file-system-access-errors.md
    ├── C1-performance-counters-issues.md
    ├── C2-os-patching-visibility.md
    ├── D1-managed-instance-activation.md
    ├── D2-hybrid-activation-errors.md
    ├── E1-iam-instance-profile.md
    ├── E2-ssm-agent-connectivity.md
    ├── F1-account-level-settings.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Registration** | A1–A2 | Managed node registration, node not showing |
| **B — Remote Access** | B1–B2 | Remote desktop failures, file system access errors |
| **C — Monitoring** | C1–C2 | Performance counters, OS patching visibility |
| **D — Activation** | D1–D2 | Managed instance activation, hybrid activation errors |
| **E — Connectivity** | E1–E2 | IAM instance profile, SSM Agent connectivity |
| **F — Settings** | F1 | Account-level settings |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `ssm:*`, `ec2:Describe*`, `iam:Get*`, `iam:ListInstanceProfiles`, `cloudtrail:LookupEvents` permissions

---

## License

MIT-0

# AWS CloudShell Diagnostics Skill

Agent skill for investigating and troubleshooting AWS CloudShell problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS CloudShell — session launches, environment persistence, network connectivity, VPC environments, storage, file transfer, permissions, credentials, timeouts, performance, and custom environments.

### Activate When

- Session launch failures
- Environment persistence issues
- Network connectivity problems
- VPC environment errors
- Storage quota or corruption issues
- File upload/download failures
- Permissions or access denied errors
- Credential or authentication issues
- Timeout or disconnection problems
- Performance degradation
- Custom environment configuration issues
- General CloudShell errors

---

## Skill Structure

```
cloudshell-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-session-launch-failures.md
    ├── A2-environment-persistence.md
    ├── B1-network-connectivity.md
    ├── B2-vpc-environment.md
    ├── C1-storage-issues.md
    ├── C2-file-transfer.md
    ├── D1-permissions-errors.md
    ├── D2-credential-issues.md
    ├── E1-timeout-disconnection.md
    ├── E2-performance-issues.md
    ├── F1-custom-environment.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Sessions** | A1–A2 | Launch failures, environment persistence |
| **B — Network** | B1–B2 | Network connectivity, VPC environment |
| **C — Storage** | C1–C2 | Storage issues, file transfer |
| **D — Permissions** | D1–D2 | Permissions errors, credential issues |
| **E — Connectivity** | E1–E2 | Timeout/disconnection, performance |
| **F — Customization** | F1 | Custom environment |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `cloudshell:*`, `iam:Get*`, `ec2:Describe*`, `cloudtrail:LookupEvents` permissions

---

## License

MIT-0

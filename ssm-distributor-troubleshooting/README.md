# AWS Systems Manager Distributor Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Systems Manager Distributor problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for SSM Distributor — package creation, version management, installation failures, uninstallation errors, association execution, scheduling, cross-platform packages, custom packages, S3 storage, IAM permissions, and third-party package issues.

### Activate When

- Package creation failures
- Package version errors
- Installation failures on managed instances
- Uninstallation errors
- Association execution failures
- Scheduling issues with package deployment
- Cross-platform package problems
- Custom package errors
- S3 package storage issues
- IAM permission errors
- Third-party package failures
- General SSM Distributor errors

---

## Skill Structure

```
ssm-distributor-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-package-creation-failures.md
    ├── A2-package-version-errors.md
    ├── B1-installation-failures.md
    ├── B2-uninstallation-errors.md
    ├── C1-association-execution-failures.md
    ├── C2-scheduling-issues.md
    ├── D1-cross-platform-packages.md
    ├── D2-custom-package-errors.md
    ├── E1-s3-package-storage.md
    ├── E2-iam-permissions.md
    ├── F1-third-party-package-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Packages** | A1–A2 | Creation failures, version errors |
| **B — Installation** | B1–B2 | Installation failures, uninstallation errors |
| **C — Associations** | C1–C2 | Execution failures, scheduling issues |
| **D — Platform & Custom** | D1–D2 | Cross-platform, custom packages |
| **E — Storage & IAM** | E1–E2 | S3 storage, IAM permissions |
| **F — Third-Party** | F1 | Third-party package issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `ssm:*`, `s3:GetObject`, `s3:ListBucket`, `iam:Get*`, `ec2:DescribeInstances`, `cloudtrail:LookupEvents` permissions

---

## License

MIT-0

# AWS Signer Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Signer problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Signer — signing profiles, signing jobs, signature validation, Lambda code signing, IoT code signing, signing platforms, certificate issues, IAM permissions, cross-account signing, and notification configuration.

### Activate When

- Signing profile creation failures
- Signing profile revocation issues
- Signing job failures
- Signature validation errors
- Lambda code signing problems
- IoT code signing issues
- Signing platform compatibility errors
- Certificate issues
- IAM permission errors for signing
- Cross-account signing failures
- Notification configuration problems
- General AWS Signer errors

---

## Skill Structure

```
signer-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-signing-profile-creation.md
    ├── A2-signing-profile-revocation.md
    ├── B1-signing-job-failures.md
    ├── B2-signature-validation-errors.md
    ├── C1-lambda-code-signing.md
    ├── C2-iot-code-signing.md
    ├── D1-signing-platform-issues.md
    ├── D2-certificate-issues.md
    ├── E1-iam-permissions.md
    ├── E2-cross-account-signing.md
    ├── F1-notification-configuration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Profiles** | A1–A2 | Signing profile creation, profile revocation |
| **B — Jobs** | B1–B2 | Signing job failures, signature validation errors |
| **C — Integration** | C1–C2 | Lambda code signing, IoT code signing |
| **D — Platform** | D1–D2 | Signing platform issues, certificate issues |
| **E — Access** | E1–E2 | IAM permissions, cross-account signing |
| **F — Notifications** | F1 | Notification configuration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `signer:*`, `lambda:GetCodeSigningConfig`, `iot:Describe*`, `iam:SimulatePrincipalPolicy`, `kms:Describe*` permissions

---

## License

MIT-0

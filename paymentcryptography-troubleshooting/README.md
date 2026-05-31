# AWS Payment Cryptography Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Payment Cryptography problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Payment Cryptography — key creation, key import/export, encryption/decryption operations, PIN operations, MAC verification, key aliases, access control, audit logging, compliance, and key rotation.

### Activate When

- Key creation failures
- Key import errors
- Encryption operation failures
- Decryption errors
- PIN operation issues
- MAC verification failures
- Key alias problems
- Key export errors
- Access control issues
- Audit logging problems
- Compliance configuration issues
- Key rotation failures
- Integration issues
- General Payment Cryptography errors

---

## Skill Structure

```
paymentcryptography-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-key-creation-failures.md
    ├── A2-key-import-errors.md
    ├── B1-encryption-operation-failures.md
    ├── B2-decryption-errors.md
    ├── C1-pin-operations.md
    ├── C2-mac-verification.md
    ├── D1-key-alias-issues.md
    ├── D2-key-export-errors.md
    ├── E1-access-control.md
    ├── E2-audit-logging.md
    ├── F1-compliance-configuration.md
    ├── F2-key-rotation.md
    ├── G1-integration-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Key Management** | A1–A2 | Creation, import |
| **B — Cryptographic Ops** | B1–B2 | Encryption, decryption |
| **C — Payment Ops** | C1–C2 | PIN operations, MAC verification |
| **D — Key Lifecycle** | D1–D2 | Aliases, export |
| **E — Security** | E1–E2 | Access control, audit logging |
| **F — Compliance** | F1–F2 | Compliance configuration, key rotation |
| **G — Integration** | G1 | Integration issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `payment-cryptography:*`, `payment-cryptography-data:*`, `iam:Get*`, `cloudtrail:LookupEvents` permissions

---

## License

MIT-0

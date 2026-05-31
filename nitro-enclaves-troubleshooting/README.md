# AWS Nitro Enclaves Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Nitro Enclaves problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Nitro Enclaves — enclave creation, image building, startup errors, vsock communication, KMS integration, attestation, memory/CPU allocation, ACM integration, cryptographic attestation, and parent instance configuration.

### Activate When

- Enclave creation failures
- Enclave image build errors
- Enclave startup problems
- Vsock communication issues
- KMS integration or attestation failures
- Memory or CPU allocation errors
- ACM for Nitro Enclaves issues
- Cryptographic attestation problems
- Parent instance configuration issues
- General Nitro Enclaves errors

---

## Skill Structure

```
nitro-enclaves-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-enclave-creation-failures.md
    ├── A2-enclave-image-build.md
    ├── B1-enclave-start-errors.md
    ├── B2-vsock-communication.md
    ├── C1-kms-integration-errors.md
    ├── C2-attestation-failures.md
    ├── D1-memory-allocation.md
    ├── D2-cpu-allocation.md
    ├── E1-acm-integration.md
    ├── E2-cryptographic-attestation.md
    ├── F1-parent-instance-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Enclave Lifecycle** | A1–A2 | Creation failures, image build |
| **B — Startup & Communication** | B1–B2 | Start errors, vsock communication |
| **C — KMS & Attestation** | C1–C2 | KMS integration, attestation |
| **D — Resource Allocation** | D1–D2 | Memory, CPU |
| **E — Integrations** | E1–E2 | ACM, cryptographic attestation |
| **F — Parent Instance** | F1 | Parent instance issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- Nitro-based EC2 instance with `EnclaveOptions.Enabled: true`
- `nitro-cli` installed on parent instance
- AWS CLI v2 with `ec2:Describe*`, `kms:Describe*`, `acm:Describe*` permissions

---

## License

MIT-0

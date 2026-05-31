# AWS CloudHSM Diagnostics Skill

Agent skill for investigating and troubleshooting AWS CloudHSM problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS CloudHSM — cluster creation and management, HSM provisioning, client connectivity, cryptographic key operations, HSM user management, backup and restore, high availability and failover, CloudHSM SDK, PKCS#11 library, JCE provider, and OpenSSL engine integration.

### Activate When

- Cluster creation or initialization failures
- HSM provisioning stuck or failed
- Client cannot connect to HSM
- Key generation or operation failures
- HSM user management errors
- Backup or restore failures
- HA failover not working
- PKCS#11 library errors
- JCE provider exceptions
- OpenSSL engine failures
- General CloudHSM errors without clear symptoms

---

## Skill Structure

```
cloudhsm-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-cluster-creation.md
    ├── A2-hsm-provisioning.md
    ├── B1-client-connectivity.md
    ├── B2-network-issues.md
    ├── C1-key-management.md
    ├── C2-key-operations.md
    ├── D1-user-management.md
    ├── D2-credential-issues.md
    ├── E1-backup-restore.md
    ├── E2-backup-failures.md
    ├── F1-high-availability.md
    ├── F2-failover-issues.md
    ├── G1-pkcs11.md
    ├── G2-jce-provider.md
    ├── H1-openssl-integration.md
    ├── H2-sdk-configuration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Cluster** | A1–A2 | Cluster creation, HSM provisioning |
| **B — Connectivity** | B1–B2 | Client connectivity, network issues |
| **C — Keys** | C1–C2 | Key management, key operations |
| **D — Users** | D1–D2 | User management, credential issues |
| **E — Backup** | E1–E2 | Backup/restore, backup failures |
| **F — HA** | F1–F2 | High availability, failover |
| **G — SDK** | G1–G2 | PKCS#11, JCE provider |
| **H — Integration** | H1–H2 | OpenSSL, SDK configuration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `cloudhsmv2:Describe*`, `cloudhsmv2:List*`, `ec2:DescribeSecurityGroups`, `ec2:DescribeSubnets`
- CloudHSM client tools installed on client instances

---

## License

MIT-0

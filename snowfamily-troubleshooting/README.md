# AWS Snow Family Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Snow Family problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Snow Family — Snowball, Snowball Edge, and Snowcone device ordering, unlocking, data transfer, NFS shares, edge compute, EC2 on Snow, S3 adapter, clustering, networking, OTA updates, return shipping, and Snowcone-specific issues.

### Activate When

- Device ordering issues
- Device unlock failures
- Data transfer errors
- NFS share problems
- Snowball Edge compute issues
- EC2 on Snow failures
- S3 adapter errors
- Clustering issues
- Network configuration problems
- Optics connectivity failures
- OTA update issues
- Device return shipping problems
- Snowcone-specific issues
- General Snow Family errors

---

## Skill Structure

```
snowfamily-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-device-ordering-issues.md
    ├── A2-device-unlock-failures.md
    ├── B1-data-transfer-errors.md
    ├── B2-nfs-share-issues.md
    ├── C1-snowball-edge-compute.md
    ├── C2-ec2-on-snow.md
    ├── D1-s3-adapter-errors.md
    ├── D2-clustering-issues.md
    ├── E1-network-configuration.md
    ├── E2-optics-connectivity.md
    ├── F1-ota-updates.md
    ├── F2-device-return-shipping.md
    ├── G1-snowcone-specific.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Device Setup** | A1–A2 | Ordering issues, unlock failures |
| **B — Data Transfer** | B1–B2 | Transfer errors, NFS shares |
| **C — Edge Compute** | C1–C2 | Snowball Edge compute, EC2 on Snow |
| **D — Storage** | D1–D2 | S3 adapter errors, clustering |
| **E — Networking** | E1–E2 | Network configuration, optics connectivity |
| **F — Maintenance** | F1–F2 | OTA updates, return shipping |
| **G — Snowcone** | G1 | Snowcone-specific issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `snowball:*`, `s3:*`, `ec2:Describe*`, `iam:Get*` permissions
- Snowball Edge client installed for device-level operations

---

## License

MIT-0

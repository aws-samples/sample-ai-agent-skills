# Amazon FSx Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon FSx problems across all file system types (Lustre, Windows File Server, ONTAP, OpenZFS) using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon FSx when the console alone isn't enough — Lustre creation and S3 integration, Windows AD and share access, ONTAP SVM and multi-protocol issues, OpenZFS volume and snapshot management, performance analysis, backup/restore, scaling operations, networking, and security configuration.

### Activate When

- Lustre file system creation failures
- Data repository association (S3 link) issues
- Lustre throughput degradation
- Windows Active Directory integration failures
- Windows share access problems
- DFS namespace configuration issues
- ONTAP SVM creation or configuration problems
- ONTAP volume management issues
- Data deduplication not working as expected
- OpenZFS volume configuration problems
- OpenZFS snapshot management issues
- Throughput limit bottlenecks
- IOPS performance issues
- High latency on file operations
- Backup job failures
- Restore operation problems
- Storage capacity scaling issues
- Throughput scaling configuration
- VPC/subnet networking problems
- Multi-AZ failover issues
- Encryption configuration errors
- Access control and permission problems

---

## Skill Structure

```
fsx-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-lustre-creation-failures.md
    ├── A2-data-repository-association.md
    ├── A3-lustre-throughput.md
    ├── B1-ad-integration.md
    ├── B2-share-access.md
    ├── B3-dfs-namespaces.md
    ├── C1-svm-issues.md
    ├── C2-volume-management.md
    ├── C3-data-deduplication.md
    ├── D1-openzfs-volume-configuration.md
    ├── D2-openzfs-snapshot-management.md
    ├── E1-throughput-limits.md
    ├── E2-iops-issues.md
    ├── E3-latency.md
    ├── F1-backup-failures.md
    ├── F2-restore-issues.md
    ├── G1-storage-capacity-scaling.md
    ├── G2-throughput-scaling.md
    ├── H1-vpc-subnet-issues.md
    ├── H2-multi-az-failover.md
    ├── I1-encryption.md
    ├── I2-access-control.md
    ├── Z1-general-troubleshooting.md
    ├── fsx-guardrails.md
    └── fsx-hallucination-patterns.yaml
```

---

## Runbook Library (28 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Lustre** | A1–A3 | Creation failures, data repository association, throughput |
| **B — Windows** | B1–B3 | AD integration, share access, DFS namespaces |
| **C — ONTAP** | C1–C3 | SVM issues, volume management, data deduplication |
| **D — OpenZFS** | D1–D2 | Volume configuration, snapshot management |
| **E — Performance** | E1–E3 | Throughput limits, IOPS, latency |
| **F — Backup** | F1–F2 | Backup failures, restore issues |
| **G — Scaling** | G1–G2 | Storage capacity, throughput scaling |
| **H — Networking** | H1–H2 | VPC/subnet, multi-AZ failover |
| **I — Security** | I1–I2 | Encryption, access control |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/fsx-guardrails.md` covering Lustre scratch vs persistent durability, Windows AD requirement, ONTAP multi-protocol behavior, OpenZFS NFS-only limitation, storage capacity one-way scaling, Lustre S3 data repository association, Windows multi-AZ failover, ONTAP SVM isolation, throughput modes and limits, backup compatibility across types, encryption at rest immutability, and network interface requirements.

---

## Investigation Workflow

1. **Triage** — Describe file systems, check CloudWatch metrics, identify file system type and configuration
2. **Deep Dive** — Examine type-specific resources (DRA, volumes, SVMs, snapshots, backups, AD)
3. **Detailed** — Inspect CloudTrail events, KMS keys, security groups, subnets, detailed performance metrics

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `fsx:*`, `ec2:Describe*`, `ds:Describe*`, `s3:GetBucketPolicy`, `kms:DescribeKey`, `iam:GetRole`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `backup:Describe*`
- CloudWatch metrics enabled for FSx
- CloudTrail logging enabled (recommended)

---

## Usage Examples

```
# List all FSx file systems
aws fsx describe-file-systems

# Describe a specific file system
aws fsx describe-file-systems --file-system-ids fs-0123456789abcdef0

# Check Lustre data repository associations
aws fsx describe-data-repository-associations --filters Name=file-system-id,Values=fs-0123456789abcdef0

# List ONTAP volumes
aws fsx describe-volumes --filters Name=file-system-id,Values=fs-0123456789abcdef0

# Check ONTAP SVMs
aws fsx describe-storage-virtual-machines --filters Name=file-system-id,Values=fs-0123456789abcdef0

# List backups
aws fsx describe-backups --filters Name=file-system-id,Values=fs-0123456789abcdef0
```

---

## License

MIT-0

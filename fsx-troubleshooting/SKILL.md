---
name: fsx-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon FSx problems
  across all file system types (Lustre, Windows File Server, ONTAP,
  OpenZFS) by analyzing file system configurations, CloudWatch metrics,
  event logs, and following structured runbooks.
  Activate when: file system creation failures, Lustre data repository
  association issues, throughput degradation, Windows Active Directory
  integration problems, share access failures, DFS namespace issues,
  ONTAP SVM configuration, volume management, data deduplication,
  OpenZFS volume or snapshot issues, performance bottlenecks (throughput,
  IOPS, latency), backup failures, restore problems, storage capacity
  scaling, throughput scaling, VPC/subnet networking, multi-AZ failover,
  encryption issues, access control problems, or the user says something
  is wrong with FSx without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with FSx, EC2, CloudWatch, CloudTrail,
  KMS, IAM, Directory Service, S3, and optionally Backup permissions.
---

# FSx Diagnostics

## When to use

Any Amazon FSx investigation where the console alone is insufficient — file system creation debugging, Lustre S3 integration, Windows AD troubleshooting, ONTAP multi-protocol issues, OpenZFS configuration, performance analysis, backup/restore, scaling operations, networking, or security configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws fsx describe-file-systems
aws fsx describe-file-systems --file-system-ids <fs-id>
aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name FreeStorageCapacity --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name DataReadBytes --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name DataWriteBytes --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws fsx describe-data-repository-associations --filters Name=file-system-id,Values=<fs-id>
aws fsx describe-volumes --filters Name=file-system-id,Values=<fs-id>
aws fsx describe-storage-virtual-machines --filters Name=file-system-id,Values=<fs-id>
aws fsx describe-snapshots --filters Name=file-system-id,Values=<fs-id>
aws fsx describe-backups --filters Name=file-system-id,Values=<fs-id>
aws ds describe-directories
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=fsx.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name DiskIopsUtilization --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name NetworkThroughputUtilization --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws kms describe-key --key-id <key-id>
aws ec2 describe-security-groups --group-ids <sg-id>
aws ec2 describe-subnets --subnet-ids <subnet-id>
```

Read `references/fsx-guardrails.md` before concluding on any FSx issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-file-systems` | Full file system details and configuration |
| `describe-data-repository-associations` | Lustre S3 data repository links |
| `describe-volumes` | ONTAP/OpenZFS volume details |
| `describe-storage-virtual-machines` | ONTAP SVM configuration |
| `describe-snapshots` | Volume snapshots (ONTAP/OpenZFS) |
| `describe-backups` | Backup details for all FSx types |
| `ds describe-directories` | Active Directory configuration (Windows) |
| `CloudWatch FSx metrics` | Performance and capacity metrics |
| `CloudTrail fsx events` | API call history and errors |

## Gotchas: Amazon FSx

- Lustre is scratch (no replication, data lost if hardware fails) or persistent (replicated within AZ). Scratch is cheaper and higher burst throughput. Persistent supports automatic failover within AZ. Neither type supports multi-AZ. Scratch file systems cannot be converted to persistent.
- Windows File Server requires Active Directory — either AWS Managed Microsoft AD or a self-managed AD. The FSx file system joins the AD domain. DNS, domain controllers, and network connectivity between FSx subnets and AD must be working. AD trust relationships can cause join failures.
- ONTAP supports multi-protocol access: NFS, SMB, and iSCSI simultaneously on the same data. SVMs (Storage Virtual Machines) act as virtual file servers with their own network identity. Each SVM can have different protocol configurations. ONTAP supports FlexClone for instant volume copies.
- OpenZFS is NFS only — no SMB or iSCSI support. It supports data compression (LZ4, ZSTD), snapshots, and clones. OpenZFS volumes can have different record sizes optimized for workload type. Copy-on-write snapshots are space-efficient.
- Storage capacity scaling is one-way — you CANNOT shrink a file system after increasing capacity. Capacity increases take time to complete and the file system remains available during scaling. For Lustre, capacity increases must be in increments of the original capacity.
- Lustre data repository association links the file system to S3. Data is lazy-loaded from S3 on first access. Export tasks push changes back to S3. Auto-import can keep the file system metadata in sync with S3 changes. DRA failures are often caused by S3 bucket policy or IAM role issues.
- Windows file system has single-AZ and multi-AZ deployment options. Multi-AZ provides automatic failover to a standby in a different AZ. Single-AZ is cheaper but has no automatic failover. Multi-AZ uses a floating IP that moves during failover. DNS updates propagate within 60 seconds.
- ONTAP SVM is like a virtual file server — each SVM has its own DNS name, network interfaces, and protocol configuration. A single ONTAP file system can host multiple SVMs. SVMs provide multi-tenancy and workload isolation. SVM administration can be delegated.

## Anti-hallucination rules

1. Always cite specific file system IDs, volume IDs, SVM IDs, or API responses as evidence.
2. Lustre scratch file systems have NO data replication. Never claim scratch Lustre is durable.
3. Storage capacity CANNOT be decreased. Never suggest shrinking a file system.
4. OpenZFS supports NFS only. Never suggest SMB or iSCSI access for OpenZFS.
5. Windows FSx REQUIRES Active Directory. Never suggest creating Windows FSx without AD.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 28 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Lustre | A1-A3 | Creation failures, data repository association, throughput |
| B — Windows | B1-B3 | AD integration, share access, DFS namespaces |
| C — ONTAP | C1-C3 | SVM issues, volume management, data deduplication |
| D — OpenZFS | D1-D2 | Volume configuration, snapshot management |
| E — Performance | E1-E3 | Throughput limits, IOPS, latency |
| F — Backup | F1-F2 | Backup failures, restore issues |
| G — Scaling | G1-G2 | Storage capacity, throughput scaling |
| H — Networking | H1-H2 | VPC/subnet, multi-AZ failover |
| I — Security | I1-I2 | Encryption, access control |
| Z — Catch-All | Z1 | General troubleshooting |

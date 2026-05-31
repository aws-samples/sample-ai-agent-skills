---
name: efs-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon EFS problems
  by analyzing file system configurations, mount targets, security
  groups, and following structured runbooks. Activate when: mount
  failures, NFS timeouts, throughput issues, burst credit depletion,
  security group blocks, access point problems, EFS CSI driver
  errors, lifecycle policy issues, replication failures, encryption
  problems, or the user says something is wrong with EFS without
  naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with EFS, EC2, IAM, CloudWatch,
  and optionally EKS permissions.
---

# EFS Diagnostics

## When to use

Any EFS investigation where the console alone is insufficient — mount failures, NFS errors, throughput bottlenecks, security group debugging, access point configuration, EKS CSI driver issues, or lifecycle policy troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws efs describe-file-systems --file-system-id <fs-id>
aws efs describe-mount-targets --file-system-id <fs-id>
aws efs describe-mount-target-security-groups --mount-target-id <mt-id>
aws efs describe-file-system-policy --file-system-id <fs-id>
aws efs describe-access-points --file-system-id <fs-id>
aws efs describe-lifecycle-configuration --file-system-id <fs-id>
```

### Step 2 — Domain deep dive

```
aws efs describe-replication-configurations --file-system-id <fs-id>
aws ec2 describe-security-groups --group-ids <sg-id>
aws cloudwatch get-metric-statistics --namespace AWS/EFS --metric-name BurstCreditBalance --dimensions Name=FileSystemId,Value=<fs-id> ...
aws cloudwatch get-metric-statistics --namespace AWS/EFS --metric-name TotalIOBytes --dimensions Name=FileSystemId,Value=<fs-id> ...
aws cloudwatch get-metric-statistics --namespace AWS/EFS --metric-name PercentIOLimit --dimensions Name=FileSystemId,Value=<fs-id> ...
```

Read `references/efs-guardrails.md` before concluding on any EFS issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-file-systems` | File system config, performance mode, throughput mode |
| `describe-mount-targets` | Mount target AZ, subnet, IP |
| `describe-mount-target-security-groups` | Security groups on mount targets |
| `describe-file-system-policy` | Resource-based policy |
| `describe-access-points` | Access point POSIX config |
| `describe-lifecycle-configuration` | IA transition rules |
| `describe-replication-configurations` | Cross-region replication |
| CloudWatch EFS metrics | Throughput, IOPS, burst credits, IO limit |

## Gotchas: EFS

- General Purpose vs Max I/O: General Purpose is recommended for most workloads. It has lower latency. Max I/O has higher aggregate throughput but higher latency. Check PercentIOLimit metric — if consistently above 95%, consider Max I/O.
- Bursting vs Provisioned vs Elastic throughput: Bursting throughput scales with file system size (50 MiB/s per TB). Small file systems run out of burst credits quickly. Provisioned allows setting throughput independently. Elastic automatically scales (most expensive).
- Mount targets: one per AZ per VPC. Instances must use the mount target in their AZ. Cross-AZ mount works but incurs data transfer costs.
- NFS port 2049 must be open in the mount target's security group. Both inbound (from client) and the client's security group must allow outbound to port 2049.
- EFS is regional, not AZ-specific. Data is replicated across AZs automatically. Mount targets provide AZ-local access points.
- Access points enforce POSIX user ID and group ID. They override the client's NFS credentials. Useful for container workloads.
- EFS CSI driver (EKS) needs mount targets in the subnets where worker nodes run. Missing mount targets cause PVC pending.
- Lifecycle policies move files to Infrequent Access (IA) storage. Minimum file size is 128 KB. Files smaller than 128 KB stay in Standard.
- Replication is asynchronous with RPO of minutes. The destination file system is read-only until you fail over.
- Encryption at rest must be enabled at file system creation. It cannot be added later. In-transit encryption is enabled via the mount helper (amazon-efs-utils).

### Throughput mode comparison

| Mode | Behavior | Best For |
|------|----------|----------|
| Bursting | 50 MiB/s per TB + burst to 100 MiB/s | Large file systems with variable load |
| Provisioned | Fixed throughput regardless of size | Small file systems needing consistent throughput |
| Elastic | Auto-scales up to 10 GiB/s | Unpredictable workloads, highest cost |

## Anti-hallucination rules

1. Always cite specific file system configurations, security group rules, or CloudWatch metrics as evidence.
2. General Purpose mode is recommended for most workloads. Never default to Max I/O without checking PercentIOLimit.
3. Mount targets are per-AZ. Never claim a single mount target serves all AZs.
4. Encryption at rest cannot be added after creation. Never suggest enabling it on an existing unencrypted file system.
5. Burst credits deplete on small file systems. Never ignore BurstCreditBalance when diagnosing throughput issues.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 24 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Mount Issues | A1-A3 | Mount failures, timeout, DNS resolution |
| B — Performance | B1-B3 | Throughput limits, latency, burst credits |
| C — Security | C1-C3 | Security group issues, IAM auth, access points |
| D — EKS Integration | D1-D2 | CSI driver, PVC issues |
| E — Lifecycle & Replication | E1-E2 | IA transitions, replication |
| F — Encryption | F1-F2 | At-rest, in-transit |
| Z — Catch-All | Z1 | General troubleshooting |

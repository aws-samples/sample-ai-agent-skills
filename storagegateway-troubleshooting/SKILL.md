---
name: storagegateway-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Storage Gateway problems
  by analyzing gateway deployment, file gateway shares, volume gateway issues,
  tape gateway operations, cache performance, networking, backup/recovery,
  security, and following structured runbooks. Activate when: gateway deployment
  failures, gateway offline, activation issues, file share problems, NFS/SMB
  share errors, cache refresh failures, volume gateway issues, cached vs stored
  volume problems, tape gateway failures, virtual tape issues, cache performance
  degradation, upload bandwidth problems, connectivity failures, firewall port
  issues, snapshot failures, recovery problems, AD integration errors, encryption
  issues, or the user says something is wrong with Storage Gateway without
  naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with StorageGateway, S3, EC2, IAM,
  CloudWatch, CloudTrail, and optionally Active Directory permissions.
  Some operations require local gateway VM access.
---

# AWS Storage Gateway Diagnostics

## When to use

Any AWS Storage Gateway investigation where the console alone is insufficient — gateway deployment and activation, file/volume/tape gateway operations, cache management, networking, backup and recovery, security configuration, or performance troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws storagegateway list-gateways --query 'Gateways[*].{Id:GatewayId,Name:GatewayName,Type:GatewayType,State:GatewayOperationalState}'
version: "1.0.0"
last_updated: "2025-04-12"
aws storagegateway describe-gateway-information --gateway-arn <gateway-arn>
aws storagegateway describe-cache --gateway-arn <gateway-arn>
aws storagegateway describe-bandwidth-rate-limit --gateway-arn <gateway-arn>
```

### Step 2 — Domain deep dive

```
aws storagegateway list-file-shares --gateway-arn <gateway-arn>
aws storagegateway describe-nfs-file-shares --file-share-arn-list <file-share-arn>
aws storagegateway describe-smb-file-shares --file-share-arn-list <file-share-arn>
aws storagegateway list-volumes --gateway-arn <gateway-arn>
aws storagegateway describe-cached-iscsi-volumes --volume-arns <volume-arns>
aws storagegateway list-tapes --gateway-arn <gateway-arn>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=storagegateway.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/StorageGateway --metric-name CacheHitPercent --dimensions Name=GatewayId,Value=<gateway-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws storagegateway describe-maintenance-start-time --gateway-arn <gateway-arn>
```

Read `references/storagegateway-guardrails.md` before concluding on any Storage Gateway issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-gateway-information` | Check gateway type, status, timezone, network |
| `describe-cache` | Check cache disk allocation and usage |
| `describe-nfs-file-shares` / `describe-smb-file-shares` | Check file share configuration |
| `describe-cached-iscsi-volumes` | Check volume gateway configuration |
| `list-tapes` / `describe-tapes` | Check tape gateway and virtual tapes |
| `describe-bandwidth-rate-limit` | Check upload/download bandwidth limits |
| CloudWatch Metrics | Monitor cache hit rate, throughput, latency |
| CloudTrail | Audit API calls and configuration changes |

## Gotchas: AWS Storage Gateway

- Storage Gateway requires local cache disks for ALL gateway types. File gateways cache recently accessed files. Volume gateways (cached mode) cache frequently accessed data. Tape gateways cache data before uploading. Insufficient cache causes performance degradation and write failures. Cache disks should be SSD for best performance.
- Gateway VM requires minimum resources that vary by type. File gateway: 4 vCPUs, 16 GB RAM. Volume/Tape gateway: 4 vCPUs, 16 GB RAM. Cache disk minimum is 150 GB. Upload buffer minimum is 150 GB. Undersized VMs cause poor performance or gateway crashes.
- File gateway NFS/SMB shares map to S3 buckets. Each file share maps to one S3 bucket (or prefix). Files are stored as S3 objects. File metadata is stored as S3 object metadata. Changes made directly to S3 objects are NOT automatically reflected in the file share — a cache refresh is required.
- Gateway activation requires port 80 temporarily. Port 80 is only needed during activation. After activation, the gateway communicates on port 443 (HTTPS). The gateway also needs ports 1026-1028, 1031, and 2222 for local console access.
- Volume gateway has two modes: cached and stored. Cached volumes store primary data in S3 with frequently accessed data cached locally. Stored volumes store primary data locally with asynchronous backup to S3. The mode is set per volume and cannot be changed after creation.
- Tape gateway virtual tapes have size limits. Individual virtual tapes can be 100 GB to 5 TB. A tape gateway can have up to 1,500 virtual tapes. Tapes must be ejected before archiving. Archived tapes go to S3 Glacier or S3 Glacier Deep Archive.
- Cache refresh for file shares is not automatic for S3 changes. If files are added/modified directly in S3 (not through the gateway), the file share cache must be refreshed manually or via scheduled refresh. Without refresh, the gateway shows stale data.
- AD integration for SMB shares requires proper DNS and network configuration. The gateway must be able to reach the AD domain controller. DNS must resolve the AD domain. Time synchronization between gateway and AD is critical.

## Anti-hallucination rules

1. Always cite specific gateway IDs, file share ARNs, or API responses as evidence.
2. Cache disks are NOT the same as upload buffer disks. Never confuse the two.
3. File gateway changes to S3 require cache refresh. Never claim auto-sync from S3.
4. Volume gateway modes (cached vs stored) cannot be changed after creation.
5. Gateway activation port (80) is temporary. Normal operation uses port 443.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 24 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Gateway | A1–A3 | Deployment failures, gateway offline, activation |
| B — File | B1–B3 | File gateway issues, NFS/SMB shares, cache refresh |
| C — Volume | C1–C2 | Volume gateway, cached vs stored volumes |
| D — Tape | D1–D2 | Tape gateway, virtual tape issues |
| E — Performance | E1–E2 | Cache performance, upload bandwidth |
| F — Networking | F1–F2 | Connectivity, firewall ports |
| G — Backup | G1–G2 | Snapshot issues, recovery |
| H — Security | H1–H2 | AD integration, encryption |
| Z — Catch-All | Z1 | General troubleshooting |

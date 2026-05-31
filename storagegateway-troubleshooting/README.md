# AWS Storage Gateway Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Storage Gateway problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Storage Gateway when the console alone isn't enough — gateway deployment and activation, file gateway NFS/SMB shares, volume gateway cached and stored modes, tape gateway virtual tapes, cache performance, upload bandwidth, networking and firewall ports, snapshot and recovery operations, Active Directory integration, and encryption configuration.

### Activate When

- Gateway deployment or activation failures
- Gateway offline or unreachable
- File share creation or access issues
- NFS/SMB share permission problems
- Cache refresh not reflecting S3 changes
- Volume gateway read/write errors
- Cached vs stored volume issues
- Tape gateway or virtual tape failures
- Cache hit rate degradation
- Upload bandwidth bottlenecks
- Gateway connectivity failures
- Firewall port configuration issues
- Snapshot creation or restore failures
- Recovery operation problems
- Active Directory integration errors
- Encryption configuration issues
- General Storage Gateway errors without clear symptoms

---

## Skill Structure

```
storagegateway-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-deployment-failures.md
    ├── A2-gateway-offline.md
    ├── A3-activation.md
    ├── B1-file-gateway-issues.md
    ├── B2-nfs-smb-shares.md
    ├── B3-cache-refresh.md
    ├── C1-volume-gateway.md
    ├── C2-cached-vs-stored.md
    ├── D1-tape-gateway.md
    ├── D2-virtual-tape-issues.md
    ├── E1-cache-performance.md
    ├── E2-upload-bandwidth.md
    ├── F1-connectivity.md
    ├── F2-firewall-ports.md
    ├── G1-snapshot-issues.md
    ├── G2-recovery.md
    ├── H1-ad-integration.md
    ├── H2-encryption.md
    ├── Z1-general-troubleshooting.md
    ├── storagegateway-guardrails.md
    └── storagegateway-hallucination-patterns.yaml
```

---

## Runbook Library (24 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Gateway** | A1–A3 | Deployment failures, gateway offline, activation |
| **B — File** | B1–B3 | File gateway issues, NFS/SMB shares, cache refresh |
| **C — Volume** | C1–C2 | Volume gateway, cached vs stored volumes |
| **D — Tape** | D1–D2 | Tape gateway, virtual tape issues |
| **E — Performance** | E1–E2 | Cache performance, upload bandwidth |
| **F — Networking** | F1–F2 | Connectivity, firewall ports |
| **G — Backup** | G1–G2 | Snapshot issues, recovery |
| **H — Security** | H1–H2 | AD integration, encryption |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/storagegateway-guardrails.md` covering cache disk requirements, gateway VM sizing, file-to-S3 object mapping, activation port requirements, volume gateway mode immutability, virtual tape size limits, cache refresh behavior, AD integration requirements, upload buffer vs cache disk distinction, gateway maintenance windows, bandwidth rate limiting, and gateway type selection.

---

## Investigation Workflow

1. **Triage** — Check gateway status, cache health, file share/volume/tape status
2. **Deep Dive** — Examine specific gateway type configuration, cache metrics, network connectivity
3. **Detailed** — Review CloudTrail events, CloudWatch metrics, gateway logs

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `storagegateway:Describe*`, `storagegateway:List*`, `s3:GetBucketLocation`, `ec2:DescribeInstances`, `cloudtrail:LookupEvents`, `cloudwatch:GetMetricStatistics`
- Local access to gateway VM for network and resource troubleshooting (optional)
- Active Directory credentials for SMB share troubleshooting (optional)

---

## Usage Examples

```
# List all gateways
aws storagegateway list-gateways

# Check gateway details
aws storagegateway describe-gateway-information --gateway-arn <gateway-arn>

# Check cache status
aws storagegateway describe-cache --gateway-arn <gateway-arn>

# List file shares
aws storagegateway list-file-shares --gateway-arn <gateway-arn>

# Check bandwidth limits
aws storagegateway describe-bandwidth-rate-limit --gateway-arn <gateway-arn>

# Refresh file share cache
aws storagegateway refresh-cache --file-share-arn <file-share-arn>
```

---

## License

MIT-0

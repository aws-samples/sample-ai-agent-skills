---
name: cloudhsm-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS CloudHSM problems
  by analyzing cluster creation, HSM provisioning, client connectivity,
  key management, user management, backup/restore, high availability,
  CloudHSM SDK, PKCS#11, JCE provider, and OpenSSL integration following
  structured runbooks. Activate when: cluster creation failures, HSM
  initialization issues, client connection errors, key operation failures,
  crypto user management problems, backup failures, HA failover issues,
  SDK errors, PKCS#11 library problems, JCE provider exceptions, OpenSSL
  engine errors, or the user says something is wrong with CloudHSM
  without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with CloudHSMv2, EC2, IAM, CloudWatch,
  CloudTrail, and VPC permissions. CloudHSM client tools for HSM-level
  operations.
---

# AWS CloudHSM Diagnostics

## When to use

Any AWS CloudHSM investigation where the console alone is insufficient — cluster and HSM management, client connectivity, cryptographic key operations, HSM user management, backup and restore, high availability, SDK integration, PKCS#11 library, JCE provider, or OpenSSL engine issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws cloudhsmv2 describe-clusters
aws cloudhsmv2 describe-clusters --filters clusterIds=<cluster-id>
aws cloudhsmv2 list-tags --resource-id <cluster-id>
```

### Step 2 — Domain deep dive

```
aws cloudhsmv2 describe-backups --filters clusterIds=<cluster-id>
aws ec2 describe-security-groups --group-ids <cluster-sg-id>
aws ec2 describe-subnets --subnet-ids <subnet-id>
# On client instance:
/opt/cloudhsm/bin/cloudhsm-cli cluster info
/opt/cloudhsm/bin/cloudhsm-cli user list
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudhsmv2.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/CloudHSM --metric-name HsmKeysSessionCount --dimensions Name=ClusterId,Value=<id> --start-time <start> --end-time <end> --period 300 --statistics Average
# Check client logs:
# /opt/cloudhsm/run/cloudhsm-client.log
```

Read `references/guardrails.md` before concluding on any CloudHSM issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-clusters` | Check cluster and HSM status |
| `describe-backups` | Check backup status and history |
| `cloudhsm-cli` | HSM-level user and key management |
| Security Groups | Verify client-to-HSM connectivity |
| CloudWatch Metrics | Monitor HSM health and usage |
| CloudTrail | Audit management API calls |
| Client logs | Debug connectivity and operation errors |

## Gotchas: AWS CloudHSM

- CloudHSM is a single-tenant HSM. You manage HSM users, keys, and access. AWS manages hardware and firmware. AWS cannot access your keys or recover them if credentials are lost. Losing all crypto officer (CO) credentials means losing key management access permanently.
- Client connectivity requires: (1) CloudHSM client installed on EC2 instance, (2) instance in same VPC or peered VPC, (3) security group allows port 2223-2225 from client to HSM ENI, (4) client configuration points to cluster IP.
- HSM users are managed INSIDE the HSM, not via IAM. Three user types: PRECO (pre-crypto officer for initial setup), CO (crypto officer — manages users), CU (crypto user — performs crypto operations). IAM controls management plane (create/delete cluster), HSM users control data plane.
- Clusters should have 2+ HSMs in different AZs for HA. Keys are automatically synchronized across HSMs in a cluster. If one HSM fails, the client SDK automatically fails over to another HSM. Single-HSM clusters have no failover.
- Backups are encrypted and stored in S3 (managed by AWS). Backups contain all HSM users and keys. Restoring a backup creates a new cluster with the same users and keys. Backups are automatic (daily) and can be triggered manually.
- PKCS#11, JCE, and OpenSSL integrations use the CloudHSM client SDK. Each has specific library paths and configuration. Version mismatches between SDK components cause failures.

## Anti-hallucination rules

1. Always cite specific cluster IDs, HSM IDs, or client log entries as evidence.
2. AWS CANNOT recover keys or reset HSM users. Never suggest contacting AWS for key recovery.
3. HSM users are NOT IAM users. Never confuse IAM and HSM user management.
4. Port 2223-2225 is required for client connectivity. Never use other ports.
5. Single-HSM clusters have NO failover. Never claim HA with one HSM.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Cluster | A1–A2 | Cluster creation, HSM provisioning |
| B — Connectivity | B1–B2 | Client connectivity, network issues |
| C — Keys | C1–C2 | Key management, key operations |
| D — Users | D1–D2 | User management, credential issues |
| E — Backup | E1–E2 | Backup/restore, backup failures |
| F — HA | F1–F2 | High availability, failover |
| G — SDK | G1–G2 | PKCS#11, JCE provider |
| H — Integration | H1–H2 | OpenSSL, SDK configuration |
| Z — Catch-All | Z1 | General troubleshooting |

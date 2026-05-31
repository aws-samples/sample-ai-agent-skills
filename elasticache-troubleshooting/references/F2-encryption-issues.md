---
title: "F2 — ElastiCache Encryption Issues"
description: "Diagnose encryption in-transit (TLS) and at-rest issues for ElastiCache clusters"
status: active
severity: HIGH
triggers:
  - "encryption"
  - "TLS"
  - "SSL"
  - "in-transit encryption"
  - "at-rest encryption"
  - "KMS"
  - "certificate"
owner: devops-agent
objective: "Resolve encryption configuration issues and ensure data protection requirements are met"
context: "ElastiCache supports encryption in-transit (TLS) and at-rest (KMS). In-transit encryption can ONLY be enabled at cluster creation — it cannot be added later. At-rest encryption uses AWS-managed keys or customer-managed KMS keys. TLS adds ~25% CPU overhead. Clients must use TLS connections when in-transit encryption is enabled. Redis uses port 6379 (non-TLS) or 6379 with TLS (same port, TLS negotiation)."
---

## Phase 1 — Triage

MUST:
- Check encryption settings: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].{TransitEncryption:TransitEncryptionEnabled,AtRestEncryption:AtRestEncryptionEnabled,KmsKeyId:KmsKeyId}'`
- Test TLS connection: `redis-cli -h <endpoint> -p 6379 --tls PING`
- Test non-TLS connection: `redis-cli -h <endpoint> -p 6379 PING`
- Check if KMS key is accessible: `aws kms describe-key --key-id <kms-key-id>` (if at-rest encryption with CMK)
- Check EngineCPU for TLS overhead: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name EngineCPUUtilization --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Verify client TLS configuration (CA certificate, TLS version)
- Check if the application framework supports TLS connections to Redis
- Verify KMS key policy allows ElastiCache service: `aws kms get-key-policy --key-id <kms-key-id> --policy-name default`
- Check for certificate expiration issues

MAY:
- Check TLS version negotiated by the client
- Review if stunnel is needed for older clients that don't support TLS natively
- Check if encryption requirements are mandated by compliance (PCI, HIPAA)

## Phase 2 — Remediate

MUST:
- If TLS is needed but not enabled, create a new cluster with TLS and migrate data (cannot enable on existing cluster)
- Ensure all clients use TLS when in-transit encryption is enabled
- For redis-cli, use: `redis-cli -h <endpoint> -p 6379 --tls --cacert /path/to/AmazonRootCA1.pem`

SHOULD:
- Use customer-managed KMS keys for at-rest encryption for key rotation control
- Account for ~25% CPU overhead when sizing nodes with TLS enabled
- Update application Redis client libraries to versions that support TLS

MAY:
- Use stunnel as a TLS proxy for clients that don't support native TLS
- Implement certificate pinning for additional security
- Enable both in-transit and at-rest encryption for defense in depth

## Common Issues

- symptoms: "Connection refused when using --tls flag"
  diagnosis: "In-transit encryption is not enabled on the cluster."
  resolution: "The cluster was created without TLS. Create a new cluster with TransitEncryptionEnabled=true."

- symptoms: "Connection works without TLS but fails with TLS"
  diagnosis: "TLS is enabled but the client certificate configuration is incorrect."
  resolution: "Ensure the client trusts the Amazon Root CA. Download AmazonRootCA1.pem."

- symptoms: "High CPU after enabling TLS on new cluster"
  diagnosis: "TLS encryption/decryption adds ~25% CPU overhead."
  resolution: "Scale up node type to accommodate TLS overhead. This is expected behavior."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create new cluster with TLS | YELLOW | Requires data migration from existing cluster |
| Update client TLS configuration | GREEN | Application-level change; no infrastructure risk |
| Use customer-managed KMS key | GREEN | Security improvement; no operational impact |
| Scale up for TLS CPU overhead | YELLOW | Requires failover for Redis with replication |
| Use stunnel as TLS proxy | YELLOW | Adds infrastructure dependency |

## Escalation Conditions

- TLS required by compliance but not enabled (requires new cluster creation)
- KMS key inaccessible causing at-rest encryption failures
- TLS overhead causing unacceptable latency on latency-sensitive workloads
- Certificate issues preventing client connections
- Encryption configuration change required on a production cluster (TLS cannot be added post-creation)

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-replication-groups` (encryption settings) | MEDIUM | Exposes encryption configuration |
| `redis-cli --tls PING` | LOW | Connectivity test only |
| `describe-key` (KMS) | MEDIUM | Exposes KMS key configuration |
| `get-key-policy` (KMS) | MEDIUM | Exposes key access policy |
| `get-metric-statistics` (EngineCPU) | LOW | Operational metrics only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to fix encryption issues
- NEVER suggest disabling AUTH when troubleshooting encryption
- NEVER suggest disabling encryption in transit to resolve TLS connection issues (security regression)
- NEVER suggest reducing node count during peak traffic
- NEVER disable at-rest encryption on a cluster that requires it for compliance

## Phase 3 — Rollback

If encryption changes cause issues:
1. TLS cannot be disabled on an existing cluster — if TLS causes issues, revert client configuration to use TLS correctly
2. If KMS key was changed, ensure the previous key is still accessible and revert the key configuration
3. If a new cluster was created with TLS, redirect traffic back to the original non-TLS cluster if migration is incomplete
4. If stunnel was deployed and causes issues, remove stunnel and revert to direct connections
5. If node type was scaled up for TLS overhead, scale back if TLS is no longer in use

## Output Format

```yaml
root_cause: "encryption — <specific_cause>"
evidence:
  - type: encryption_config
    content: "<transit and at-rest encryption settings>"
  - type: tls_test
    content: "<TLS connection test result>"
  - type: kms_key
    content: "<KMS key status>"
severity: HIGH
mitigation:
  immediate: "Fix client TLS configuration or KMS key access"
  long_term: "Plan for TLS overhead in capacity, implement key rotation"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest disabling AUTH"
  - "NEVER suggest public subnet placement"

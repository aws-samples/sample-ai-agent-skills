---
title: "H2 — Encryption Issues"
description: "Diagnose encryption at rest and in transit issues on DocumentDB"
status: active
severity: HIGH
triggers:
  - "encryption error"
  - "KMS key"
  - "encryption at rest"
  - "TLS encryption"
  - "data encryption"
owner: devops-agent
objective: "Identify and resolve encryption configuration issues on DocumentDB"
context: "DocumentDB encrypts data at rest using KMS keys (specified at cluster creation, immutable). In-transit encryption (TLS) is enabled by default. Issues include KMS key access, key rotation, and TLS configuration."
---

## Phase 1 — Triage

MUST:
- Check cluster encryption configuration: `aws docdb describe-db-clusters --db-cluster-identifier <cluster-id>`
- Verify KMS key status: `aws kms describe-key --key-id <key-id>`
- Check TLS parameter: `aws docdb describe-db-cluster-parameters --db-cluster-parameter-group-name <param-group> --filters Name=ParameterName,Values=tls`
- Verify KMS key policy grants DocumentDB access

SHOULD:
- Check KMS key grants: `aws kms list-grants --key-id <key-id>`
- Verify key rotation status: `aws kms get-key-rotation-status --key-id <key-id>`
- Check for KMS-related errors in CloudTrail
- Verify snapshot encryption configuration

MAY:
- Review cross-region KMS key configuration for global clusters
- Check KMS key usage metrics
- Verify compliance requirements for encryption

## Phase 2 — Remediate

MUST:
- Re-enable KMS key if disabled
- Cancel key deletion if pending
- Update KMS key policy to grant rds.amazonaws.com access

SHOULD:
- Enable automatic KMS key rotation for customer-managed keys
- Keep TLS enabled for production clusters
- Document encryption configuration

MAY:
- Set up CloudWatch alarms for KMS key state changes
- Implement encryption compliance monitoring

## Common Issues

- symptoms: "Cluster inaccessible after KMS key disabled"
  diagnosis: "The KMS key used for encryption at rest was disabled or deleted."
  resolution: "Re-enable the KMS key. If deleted, restore from backup to a new cluster with a valid key."

- symptoms: "Cannot change encryption key on existing cluster"
  diagnosis: "Encryption key is set at cluster creation and cannot be changed."
  resolution: "Create a new cluster with the desired KMS key. Migrate data using snapshot restore or DMS."

- symptoms: "Snapshot copy fails with KMS error"
  diagnosis: "Cross-region snapshot copy requires a KMS key in the destination region."
  resolution: "Specify a KMS key in the destination region when copying the snapshot."

## Output Format

```yaml
root_cause: "encryption — <specific_cause>"
evidence:
  - type: encryption_config
    content: "<cluster encryption and KMS key configuration>"
  - type: kms_key
    content: "<key status, policy, and grants>"
  - type: tls
    content: "<TLS parameter configuration>"
severity: HIGH
mitigation:
  immediate: "Fix KMS key status or policy"
  long_term: "Implement encryption key management best practices"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — Encryption protects all data at rest and in transit |
| Blast Radius | CRITICAL — KMS key loss makes the entire cluster inaccessible |
| Rollback Complexity | HIGH — Encryption key cannot be changed on existing clusters |
| Compliance Scope | CRITICAL — Encryption is a mandatory compliance control |

## Escalation Conditions

- Escalate IMMEDIATELY to security if KMS key is disabled or pending deletion
- Escalate to compliance if encryption requirements are not met
- Escalate to AWS Support if encryption-related errors make the cluster inaccessible

## Data Sensitivity

- **KMS key** is the master encryption key for all cluster data — protect at all costs
- **TLS certificates** protect data in transit — manage lifecycle carefully
- **Key grants** define who can access encrypted data

## PROHIBITED Actions

- Do NOT disable or schedule deletion of KMS keys used by active clusters
- Do NOT disable TLS on production clusters
- Do NOT create production clusters without encryption at rest
- Do NOT share KMS key IDs outside the security team

## Phase 3 — Rollback

- If KMS key was disabled, re-enable immediately: `aws kms enable-key --key-id <id>`
- If KMS key deletion is pending, cancel: `aws kms cancel-key-deletion --key-id <id>`
- If key is permanently lost, restore from backup to a new cluster with a new KMS key
- Update TLS certificates if they were rotated incorrectly

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
  - "NEVER suggest disabling TLS"
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"

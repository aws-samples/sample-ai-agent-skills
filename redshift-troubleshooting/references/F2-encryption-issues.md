---
title: "F2 — Encryption Issues"
description: "Diagnose and resolve Redshift encryption configuration problems"
status: active
severity: HIGH
triggers:
  - "encryption"
  - "KMS"
  - "encrypted cluster"
  - "HSM"
  - "encryption at rest"
  - "SSL"
owner: devops-agent
objective: "Fix encryption configuration for Redshift clusters"
context: "Redshift supports encryption at rest using AWS KMS or CloudHSM. Encryption is set at cluster creation and cannot be changed without migration. SSL/TLS encrypts data in transit. COPY/UNLOAD can use client-side or server-side encryption for S3 data."
---

## Phase 1 — Triage

MUST:
- Check cluster encryption status: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].{encrypted:Encrypted,kmsKeyId:KmsKeyId}'`
- Check KMS key status: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.{state:KeyState,enabled:Enabled}'`
- Check SSL configuration: `aws redshift describe-cluster-parameters --parameter-group-name <param-group> --query "Parameters[?ParameterName=='require_ssl'].ParameterValue"`
- Verify KMS key policy allows Redshift: `aws kms get-key-policy --key-id <key-id> --policy-name default`

SHOULD:
- Check if KMS key is scheduled for deletion
- Verify cross-account KMS key access if using shared keys
- Check for KMS key rotation status

MAY:
- Review encryption audit trail in CloudTrail
- Check HSM configuration if using CloudHSM
- Verify COPY/UNLOAD encryption options

## Phase 2 — Remediate

MUST:
- If KMS key disabled: re-enable the key: `aws kms enable-key --key-id <key-id>`
- If KMS key scheduled for deletion: cancel deletion: `aws kms cancel-key-deletion --key-id <key-id>`
- If KMS key policy missing Redshift: update key policy to grant Redshift access
- If SSL required but client not using SSL: configure client to use SSL connection

SHOULD:
- Enable KMS key rotation for compliance
- Use require_ssl = true in parameter group for all connections
- Document KMS key dependencies for disaster recovery

MAY:
- Migrate unencrypted cluster to encrypted (requires snapshot, restore with encryption)
- Implement client-side encryption for sensitive COPY/UNLOAD data
- Set up KMS key monitoring alarms

## Common Issues

- symptoms: "Cluster inaccessible after KMS key was disabled"
  diagnosis: "Redshift cannot decrypt data without the KMS key."
  resolution: "Re-enable the KMS key immediately. Never disable or delete KMS keys used by active clusters."

- symptoms: "COPY fails with KMS access denied"
  diagnosis: "IAM role lacks kms:Decrypt permission for S3 objects encrypted with KMS."
  resolution: "Add kms:Decrypt and kms:DescribeKey to the IAM role for the S3 encryption key."

## Output Format

```yaml
root_cause: "Encryption issue — <specific_cause>"
evidence:
  - type: kms_key
    content: "<KMS key state and policy>"
severity: HIGH
mitigation:
  immediate: "Re-enable KMS key or fix key policy"
  long_term: "Enable key rotation, document key dependencies, set up monitoring"
```


## Safety Ratings
```
safety_ratings:
  - "Check cluster encryption status: GREEN — read-only API call"
  - "Check KMS key status: GREEN — read-only API call"
  - "Check SSL configuration: GREEN — read-only parameter inspection"
  - "Re-enable disabled KMS key: GREEN — restores access, non-destructive"
  - "Cancel KMS key deletion: GREEN — prevents permanent key loss"
  - "Update KMS key policy: YELLOW — changes who can use the encryption key"
  - "Migrate unencrypted to encrypted: RED — requires snapshot, restore, and connection string update"
```

## Escalation Conditions
- Cluster serves production analytics
- KMS key disabled or scheduled for deletion (cluster inaccessible)
- Encryption migration required for compliance
- Cross-account KMS key access issues
- SSL enforcement affecting client connectivity

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "KMS key ID and policy: encryption key configuration"
    - "Encryption status: compliance-relevant information"
    - "SSL configuration: transport security settings"
    - "User credentials: database access information"
  handling: "KMS key IDs and policies are highly sensitive. Never expose key policies externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER disable or delete a KMS key used by an active cluster
- NEVER disable SSL (require_ssl) on production clusters
- NEVER share KMS key policies or key IDs externally

## Phase 3 — Rollback
- If KMS key was re-enabled: no rollback needed — restoring access is non-destructive
- If KMS key policy was changed: restore previous key policy from CloudTrail history
- If SSL was enforced (require_ssl=true): revert parameter to false if clients cannot support SSL (not recommended)
- If encryption migration was started: restore from the original unencrypted snapshot if migration fails

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
  - "NEVER suggest disabling encryption"

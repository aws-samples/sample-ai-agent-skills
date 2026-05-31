---
title: "C3 — Client-Side Encryption Issues"
description: "Diagnose client-side encryption problems with S3 objects"
status: active
severity: MEDIUM
triggers:
  - "Client-side encryption"
  - "Cannot decrypt object"
  - "Encryption SDK"
  - "Wrong encryption key"
owner: devops-agent
objective: "Identify and resolve client-side encryption key management and compatibility issues"
context: "Client-side encryption encrypts data before sending to S3. The application manages keys. Common issues include lost encryption keys, SDK version mismatches, and metadata corruption. S3 stores encryption metadata in object metadata (x-amz-meta-x-amz-key, x-amz-meta-x-amz-matdesc)."
---

## Phase 1 — Triage

MUST:
- Check object metadata for encryption info: `aws s3api head-object --bucket <bucket> --key <key>`
- Look for x-amz-meta-x-amz-key, x-amz-meta-x-amz-iv, x-amz-meta-x-amz-matdesc in metadata
- Identify the encryption SDK and version used (AWS Encryption SDK, S3 Encryption Client)
- Verify the encryption key or KMS key is still available

SHOULD:
- Check if the object was encrypted with a different SDK version
- Verify the key material matches what was used during encryption
- Check if the material description (matdesc) matches the expected key provider

MAY:
- Check if the object was copied or moved (metadata may be lost)
- Verify the encryption algorithm matches between encrypt and decrypt

## Phase 2 — Remediate

MUST:
- Ensure the correct encryption key is available for decryption
- Use the same SDK version and configuration for decryption as was used for encryption
- If the key is in KMS, verify KMS key access (see C1)

SHOULD:
- Migrate from V1 to V2 encryption client if using deprecated APIs
- Store key metadata alongside objects for key rotation tracking
- Use KMS-managed keys for client-side encryption to simplify key management

MAY:
- Re-encrypt objects with a new key if the old key is compromised
- Implement key rotation with the AWS Encryption SDK's keyring approach

## Common Issues

- symptoms: "Cannot decrypt object — key mismatch"
  diagnosis: "The object was encrypted with a different key than what is being used for decryption."
  resolution: "Check the material description in object metadata to identify the correct key."

- symptoms: "Decryption fails after SDK upgrade"
  diagnosis: "V2 encryption client cannot decrypt V1 objects by default (security improvement)."
  resolution: "Configure V2 client with legacy decryption mode, then re-encrypt objects with V2."

- symptoms: "Object metadata lost after copy"
  diagnosis: "Copy operation did not preserve encryption metadata."
  resolution: "Use --metadata-directive COPY when copying, or re-encrypt after copy."

## Output Format

```yaml
root_cause: "client_side_encryption — <specific_cause>"
evidence:
  - type: object_metadata
    content: "<encryption metadata>"
  - type: sdk_version
    content: "<SDK and version>"
severity: MEDIUM
mitigation:
  immediate: "Restore correct key or fix SDK configuration"
  long_term: "Migrate to KMS-managed client-side encryption and V2 SDK"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟢 GREEN | Primarily diagnostic — uses head-object to check encryption metadata. Remediation focuses on application-side SDK configuration and key management, not bucket-level security changes. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Bucket contains sensitive/regulated data (PII, PHI, financial)
- Cross-account access changes are needed
- Encryption configuration changes affect multiple consumers

## Rollback
1. Before any bucket policy change: Save current policy with `aws s3api get-bucket-policy`
2. Before ACL changes: Save current ACL with `aws s3api get-bucket-acl`
3. After change: Verify access works without granting excessive permissions
4. If change causes issues: Restore the saved policy/ACL immediately
5. Cleanup: Remove any temporary access grants

## Data Sensitivity
| Command | Sensitivity | Handling |
|---------|------------|----------|
| `get-bucket-policy` | HIGH | Contains access rules — redact principals |
| `get-bucket-acl` | MEDIUM | Shows grantees — summarize |
| `get-public-access-block` | MEDIUM | Security posture — safe to include |
| `list-objects` | LOW | Object keys only — safe to include |

## Prohibited Actions
- NEVER suggest disabling S3 Block Public Access as a remediation
- NEVER suggest `"Principal": "*"` in bucket policy without Condition keys
- NEVER suggest removing bucket encryption to fix access issues
- NEVER suggest making a bucket public to resolve CORS or access issues
- NEVER suggest `s3:*` in any IAM or bucket policy fix
- ALWAYS use least-privilege: grant only the specific S3 action needed
- ALWAYS check both account-level AND bucket-level Block Public Access

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
  - command: "get-bucket-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-bucket-acl"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-public-access-block"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling S3 Block Public Access"
  - "NEVER suggest Principal: * without Condition keys"
  - "NEVER suggest removing bucket encryption"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback

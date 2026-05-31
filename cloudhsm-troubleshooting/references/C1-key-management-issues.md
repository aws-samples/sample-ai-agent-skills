---
title: "C1 — Key Management Issues"
description: "Diagnose and resolve CloudHSM key management and key operation failures"
status: active
severity: CRITICAL
triggers:
  - "key generation failed"
  - "key not found"
  - "Cfm3 error"
  - "key operation failed"
owner: devops-agent
objective: "Resolve key management issues and restore cryptographic operations on CloudHSM"
context: "CloudHSM key management involves generating, storing, and using cryptographic keys within the HSM. Keys are owned by CU (Crypto User) accounts and can be shared with other CUs. Key operations can fail due to permission issues, key attribute conflicts, HSM capacity limits, or synchronization problems across HSMs in a cluster."
---

## Phase 1 — Triage

MUST:
- List keys visible to the current user in KMU: `findKey` (in key_mgmt_util)
- Check key attributes: `getAttribute <key-handle> <attribute-file>` (in KMU)
- Verify the CU user has ownership or sharing access to the key: `getKeyInfo <key-handle>` (in KMU)
- Check HSM cluster health: `aws cloudhsmv2 describe-clusters --filters clusterIds=<cluster-id> --query 'Clusters[0].Hsms[*].{ID:HsmId,State:State}'`
- Verify the user is logged in as CU: `info` (in CMU to check login status)

SHOULD:
- Check if the key exists on all HSMs in the cluster (synchronization): `findKey` on each HSM
- Verify key type and algorithm match the intended operation
- Check CloudHSM client logs for detailed error messages: `sudo journalctl -u cloudhsm-client --no-pager -n 100`

MAY:
- List all users and their key counts: `listUsers` (in CMU)
- Check HSM capacity for key storage
- Review application logs for specific Cfm3 error codes

## Phase 2 — Remediate

MUST:
- If key not found: verify the correct CU is logged in and the key handle is valid
- If permission denied: share the key with the requesting CU using CO account: `shareKey <key-handle> <user-id> 1` (in KMU)
- If key generation fails: verify the HSM has capacity and the key parameters are valid
- If synchronization issue: the cluster will auto-sync; wait and retry the operation

SHOULD:
- For persistent key not found errors, check if the key was deleted or if the HSM was restored from a backup that predates key creation
- Verify key attributes (extractable, persistent, etc.) match the intended use case
- Test key operations with a non-production key first

MAY:
- Create a backup of the cluster before making key changes: `aws cloudhsmv2 create-cluster-backup --cluster-id <cluster-id>` (if not using automatic backups)
- Document key handles and their purposes for operational reference
- Implement key rotation procedures for long-lived keys

## Common Issues

- symptoms: "findKey returns empty list but keys should exist"
  diagnosis: "Logged in as wrong CU, or keys were not shared with this user."
  resolution: "Log in as the key owner CU, or have a CO share the keys with the current CU."

- symptoms: "genSymKey or genRSAKeyPair returns Cfm3GenerateKeyPair error"
  diagnosis: "Invalid key parameters, HSM capacity full, or insufficient permissions."
  resolution: "Verify key size and type are supported. Check HSM capacity. Ensure CU has key generation permissions."

- symptoms: "Key works on one HSM but not another"
  diagnosis: "Key synchronization lag between HSMs in the cluster."
  resolution: "Wait for automatic synchronization (usually seconds). If persistent, check HSM health."

- symptoms: "Cannot delete a key — operation returns error"
  diagnosis: "Key is in use by an application or the user lacks ownership."
  resolution: "Stop applications using the key, then delete as the key owner CU."

## Output Format

```yaml
root_cause: "Key management issue — <specific_cause>"
evidence:
  - type: key_info
    content: "<getKeyInfo or findKey output>"
  - type: error_details
    content: "<Cfm3 error code and message>"
severity: CRITICAL
mitigation:
  immediate: "Fix key permissions/attributes or regenerate the key"
  long_term: "Implement key lifecycle management and regular key audits"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (findKey/getAttribute/info): GREEN — read-only"
  - "Phase 2 key sharing: YELLOW — changes key access permissions"
  - "Phase 2 key deletion: RED — permanently destroys cryptographic material"

## Escalation Conditions

escalation_conditions:
  - "Key material may have been compromised or accessed by unauthorized users"
  - "Production encryption/decryption operations are failing"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Key synchronization failure persists across the cluster"
  - "Issue involves regulatory-compliance-critical keys (PCI, HIPAA)"

## Data Sensitivity

data_sensitivity:
  - command: "findKey"
    sensitivity: HIGH
    contains: "Key handles and key metadata"
  - command: "getKeyInfo"
    sensitivity: HIGH
    contains: "Key ownership and sharing information"
  - command: "getAttribute"
    sensitivity: HIGH
    contains: "Key attributes including type, size, and usage flags"

## Prohibited Actions

prohibited_actions:
  - "NEVER extract or export private keys from the HSM without explicit authorization"
  - "NEVER delete production keys without confirmed backup and stakeholder approval"
  - "NEVER share keys with temporary or test CU accounts in production"
  - "NEVER log key handles alongside their purpose in unsecured locations"
  - "NEVER use extractable=true for production master keys"

## Phase 3 — Rollback

- Before any key modifications, document the key handle, owner, and shared users
- If key sharing was changed incorrectly, revoke sharing: `shareKey <key-handle> <user-id> 0` (in KMU)
- If a key was accidentally deleted, restore from the most recent cluster backup
- If key generation created unwanted keys, delete them to free HSM capacity
- Verify all applications using affected keys are functioning after changes

---
title: "E3 — Multi-Region Key Issues"
description: "Diagnose multi-region key replication and usage problems"
status: active
severity: MEDIUM
triggers:
  - "multi-region key"
  - "mrk-"
  - "replicate key"
  - "cross-region encryption"
owner: devops-agent
objective: "Configure and troubleshoot multi-region key replication and usage"
context: "Multi-region keys share key material across regions but are separate KMS keys with separate ARNs, policies, and grants. The primary key can be replicated to other regions. Each replica must have its own key policy configured. Multi-region keys have a mrk- prefix in their key ID."
---

## Phase 1 — Triage

MUST:
- Describe the key and check multi-region config: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.{MultiRegion:MultiRegionConfiguration,State:KeyState}'`
- Verify the key is a multi-region key (key ID starts with mrk-)
- Check replica status in target regions
- Verify key policies in each region where the key is replicated

SHOULD:
- List all replicas: check MultiRegionConfiguration.ReplicaKeys in describe-key output
- Verify the primary key region
- Check that aliases are configured in each replica region

MAY:
- Test encrypt in one region and decrypt in another
- Check for region-specific grant requirements

## Phase 2 — Remediate

MUST:
- Create replicas in needed regions: `aws kms replicate-key --key-id <key-id> --replica-region <region>`
- Configure key policies for each replica independently
- Verify cross-region encrypt/decrypt works

SHOULD:
- Create consistent aliases across all replica regions
- Document which regions have replicas
- Configure grants in each region as needed

MAY:
- Update the primary region if needed: `aws kms update-primary-region`
- Set up monitoring for replica key health

## Common Issues

- symptoms: "Cannot decrypt in another region — key not found"
  diagnosis: "The multi-region key has not been replicated to the target region."
  resolution: "Replicate the key to the target region using replicate-key."

- symptoms: "Replica key has different permissions than primary"
  diagnosis: "Each replica has its own key policy that must be configured independently."
  resolution: "Configure the replica's key policy to match the desired access pattern."

- symptoms: "Cannot replicate key — not a multi-region key"
  diagnosis: "The key was created as a single-region key. Single-region keys cannot be converted."
  resolution: "Create a new multi-region key and re-encrypt data."

## Output Format

```yaml
root_cause: "multi_region_key_issue — <specific_cause>"
evidence:
  - type: key_metadata
    content: "<multi-region configuration and replicas>"
  - type: key_policies
    content: "<policies in each region>"
severity: MEDIUM
mitigation:
  immediate: "Replicate key to needed regions and configure policies"
  long_term: "Document multi-region key topology and maintain consistent policies"
```


## Safety Ratings

- `describe-key`: GREEN — Read-only key metadata and multi-region configuration retrieval
- `get-key-policy`: GREEN — Read-only retrieval of key policy document
- `list-grants`: GREEN — Read-only enumeration of grants
- `replicate-key`: RED — Extends key access to additional regions; affects encryption operations across multiple AWS regions
- `update-primary-region`: RED — Changes the primary key region; affects all replicas
- `put-key-policy` (replica): YELLOW — Policy changes affect access in the specific replica region

## Escalation Conditions

- Remediation requires modifying a KMS key policy
- Fix involves granting cross-account key access
- Key is used for encryption of production data (S3, EBS, RDS)
- Key state change affects multiple services
- Multi-region key changes affect encryption operations in multiple regions simultaneously
- Primary key region change is being considered

## Rollback

- Pre-change: "Save current key policy before modification"
- Verification: "Test encrypt/decrypt with the specific service after policy change"
- Revert: "Restore previous key policy if change breaks dependent services"

## Data Sensitivity

- HIGH: "Key policies reveal all authorized principals and services"
- HIGH: "Grant details expose cross-account relationships"
- HIGH: "Multi-region key configuration reveals infrastructure topology across regions"
- MEDIUM: "Key metadata reveals encryption strategy"

## Prohibited Actions

- NEVER suggest `"kms:*"` in a key policy
- NEVER suggest making a key policy public with `"Principal": "*"`
- NEVER suggest disabling key rotation as a fix
- NEVER suggest scheduling key deletion without confirming no active usage
- NEVER suggest creating a new key to bypass policy issues (data becomes inaccessible)

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
  - command: "get-key-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-key"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-grants"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest kms:* on key policies"
  - "NEVER suggest disabling key rotation"
  - "NEVER suggest making key policies public"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback

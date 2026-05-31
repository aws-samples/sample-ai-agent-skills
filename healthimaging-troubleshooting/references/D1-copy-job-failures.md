---
title: "D1 — Copy Job Failures"
description: "Diagnose image set copy operation failures"
status: active
severity: HIGH
triggers: ["copy job failed", "copy-image-set error", "cross-datastore copy failure"]
owner: devops-agent
objective: "Resolve image set copy job failures"
context: "Copy operations are asynchronous and create independent copies. Source and destination datastores can use different KMS keys."
---

## Phase 1 — Triage

MUST:
- Check copy job status: `aws medical-imaging get-image-set --datastore-id <destination-datastore-id> --image-set-id <image-set-id>`
- Verify source image set exists and is ACTIVE
- Verify IAM permissions for both source and destination datastores

SHOULD:
- Check KMS key permissions for both source (decrypt) and destination (encrypt) datastores
- Verify both datastores are in ACTIVE status

MAY:
- Review CloudTrail for CopyImageSet events
- Check service quotas for concurrent copy operations

## Phase 2 — Remediate

MUST:
- Fix IAM permissions: medical-imaging:GetImageSet on source, medical-imaging:CopyImageSet on destination
- Fix KMS permissions: kms:Decrypt on source key, kms:GenerateDataKey on destination key
- Retry copy after fixing permissions

SHOULD:
- Implement copy job monitoring with status polling
- Use exponential backoff for retry logic

MAY:
- Automate bulk copy operations with scripts
- Set up CloudWatch Events for copy job completion notifications

## Safety Ratings

```
safety_ratings:
  - "get-image-set: GREEN — read-only status check"
  - "copy-image-set: YELLOW — creates new image set in destination, consumes storage"
  - "KMS grant modifications: RED — changes encryption access"
  - "Bulk copy scripts: YELLOW — creates multiple image sets, significant storage impact"
```

## Escalation Conditions

- Copy jobs fail consistently with internal service errors
- Cross-account copy requires KMS key sharing across accounts
- Large image sets (>1GB) timeout during copy
- Copy job creates corrupted or incomplete image sets in destination

## Data Sensitivity

- Source image set: HIGH — contains medical imaging data with PHI
- Destination image set: HIGH — copy contains same PHI as source
- Copy job metadata: LOW — contains job IDs and status
- KMS key operations: MEDIUM — reveals encryption configuration

## Prohibited Actions

- NEVER copy PHI data to datastores without proper encryption
- NEVER share KMS keys across accounts without security team approval
- NEVER delete source image sets before verifying copy integrity
- NEVER run bulk copy operations without estimating storage costs

## Phase 3 — Rollback

1. If copy created incorrect image sets: delete from destination: `aws medical-imaging delete-image-set --datastore-id <dest-id> --image-set-id <image-set-id>`
2. If KMS grants were created for copy: revoke unnecessary grants after completion
3. If bulk copy was initiated in error: cancel remaining operations and delete completed copies
4. Verify source image sets are unaffected by the copy operation
5. Document copy operations for data lineage tracking

## Common Issues

- symptoms: "CopyImageSet returns AccessDeniedException"
  diagnosis: "IAM role lacks permissions on source or destination datastore."
  resolution: "Grant medical-imaging:GetImageSet on source and medical-imaging:CopyImageSet on destination. Add KMS permissions for both keys."

- symptoms: "Copy job completes but destination image set is missing frames"
  diagnosis: "Source image set version changed during copy."
  resolution: "Specify the exact version ID when initiating copy. Retry with the latest stable version."

## Output Format

```yaml
root_cause: "copy_job — <specific_cause>"
evidence:
  - type: copy_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix copy issue and retry"
  long_term: "Implement copy monitoring and validation"
```

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"

---
title: "D2 — Deletion Issues"
description: "Diagnose image set and datastore deletion problems"
status: active
severity: HIGH
triggers: ["cannot delete image set", "deletion failed", "delete-image-set error", "datastore deletion"]
owner: devops-agent
objective: "Resolve image set and datastore deletion issues"
context: "Deletion is a multi-step process. Deleted image sets may be recoverable within a retention window. Datastores must be empty before deletion."
---

## Phase 1 — Triage

MUST:
- Check image set status: `aws medical-imaging get-image-set --datastore-id <datastore-id> --image-set-id <image-set-id>`
- Verify IAM permissions for medical-imaging:DeleteImageSet
- For datastore deletion: verify datastore is empty (no image sets remain)

SHOULD:
- Check if image set is referenced by other operations (active copy jobs)
- Verify the correct version ID is being used for deletion

MAY:
- Review CloudTrail for DeleteImageSet events
- Check if retention policies affect deletion timing

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for deletion operations
- For datastore deletion: delete all image sets first, then delete datastore
- Use correct image set version ID in delete requests

SHOULD:
- Implement deletion confirmation workflows to prevent accidental deletion
- Maintain audit trail of all deletion operations

MAY:
- Automate cleanup of test/development datastores
- Implement soft-delete patterns in application layer

## Safety Ratings

```
safety_ratings:
  - "get-image-set: GREEN — read-only status check"
  - "delete-image-set: RED — permanently removes medical imaging data"
  - "delete-datastore: RED — permanently removes datastore and configuration"
  - "Bulk deletion scripts: RED — mass data removal, high risk"
```

## Escalation Conditions

- Deletion of PHI data requires compliance team approval
- Datastore deletion blocked by remaining image sets that cannot be identified
- Deletion required for regulatory compliance (data retention expiry)
- Accidental deletion of production image sets

## Data Sensitivity

- `aws medical-imaging get-image-set`: LOW — returns image set metadata
- Deletion operations: HIGH — permanently removes PHI data
- Deletion audit logs: MEDIUM — records what was deleted and by whom

## Prohibited Actions

- NEVER delete image sets containing active clinical data without explicit approval
- NEVER delete datastores without verifying all data has been migrated or backed up
- NEVER run bulk deletion scripts in production without a dry-run first
- NEVER bypass deletion confirmation workflows

## Phase 3 — Rollback

1. If image set was deleted within retention window: attempt recovery (if supported)
2. If datastore was deleted: data cannot be recovered — escalate immediately
3. If bulk deletion removed unintended image sets: restore from backup or re-import from S3
4. Document all deletions for regulatory compliance audit
5. Notify clinical and compliance teams of any unintended data loss

## Common Issues

- symptoms: "DeleteImageSet returns ConflictException"
  diagnosis: "Image set has an active copy job or is being modified."
  resolution: "Wait for active operations to complete. Check for in-progress copy jobs or metadata updates."

- symptoms: "DeleteDatastore returns ConflictException"
  diagnosis: "Datastore still contains image sets."
  resolution: "Delete all image sets first: list with search-image-sets, then delete each. Retry datastore deletion after all image sets are removed."

## Output Format

```yaml
root_cause: "deletion — <specific_cause>"
evidence:
  - type: deletion_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Resolve deletion blocker"
  long_term: "Implement deletion workflows with approval gates"
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

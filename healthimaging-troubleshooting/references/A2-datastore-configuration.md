---
title: "A2 — Datastore Configuration"
description: "Diagnose datastore configuration and management issues"
status: active
severity: MEDIUM
triggers: ["datastore configuration", "datastore settings", "datastore status"]
owner: devops-agent
objective: "Resolve HealthImaging datastore configuration issues"
context: "Datastores have limited mutable configuration. Encryption and region are immutable after creation."
---

## Phase 1 — Triage

MUST:
- Get datastore details: `aws medical-imaging get-datastore --datastore-id <datastore-id>`
- Verify datastore status is ACTIVE
- Check datastore tags: `aws medical-imaging list-tags-for-resource --resource-arn <datastore-arn>`

SHOULD:
- Verify KMS key status if customer-managed: `aws kms describe-key --key-id <key-id>`
- Check CloudWatch metrics for the datastore

MAY:
- Review datastore creation history in CloudTrail

## Phase 2 — Remediate

MUST:
- If datastore is in CREATE_FAILED status: delete and recreate with correct configuration
- If KMS key is disabled or pending deletion: re-enable or cancel deletion

SHOULD:
- Update tags for proper resource management: `aws medical-imaging tag-resource --resource-arn <arn> --tags Key=Environment,Value=Production`

MAY:
- Set up CloudWatch alarms for datastore metrics

## Safety Ratings

```
safety_ratings:
  - "get-datastore: GREEN — read-only datastore details"
  - "list-tags-for-resource: GREEN — read-only tag enumeration"
  - "tag-resource: GREEN — adds metadata tags, non-destructive"
  - "delete-datastore: RED — permanently removes datastore and all contents"
```

## Escalation Conditions

- Datastore stuck in CREATING or DELETING status for extended period
- KMS key used by datastore is scheduled for deletion
- Datastore needs encryption change requiring data migration
- Datastore status shows unexpected errors

## Data Sensitivity

- `aws medical-imaging get-datastore`: LOW — returns configuration metadata
- `aws medical-imaging list-tags-for-resource`: LOW — returns tag metadata
- `aws kms describe-key`: LOW — returns key metadata only

## Prohibited Actions

- NEVER delete a datastore without confirming all image sets have been migrated
- NEVER disable or schedule deletion of a KMS key used by an active datastore
- NEVER assume datastore configuration can be changed after creation for encryption settings
- NEVER modify tags used for compliance tracking without approval

## Phase 3 — Rollback

1. If tags were modified incorrectly: restore original tags using tag-resource and untag-resource
2. If KMS key was re-enabled unnecessarily: verify it should remain enabled for the datastore
3. If a datastore was deleted prematurely: data cannot be recovered — escalate to compliance team
4. Document all configuration changes and notify the operations team
5. Verify datastore is functioning normally after any changes

## Common Issues

- symptoms: "Datastore status is CREATE_FAILED"
  diagnosis: "Creation failed due to KMS key permissions or service quota."
  resolution: "Delete the failed datastore, fix the root cause, and recreate."

- symptoms: "Datastore operations return KMS errors"
  diagnosis: "Customer-managed KMS key is disabled or pending deletion."
  resolution: "Re-enable the KMS key or cancel scheduled deletion: aws kms enable-key --key-id <key-id>."

## Output Format

```yaml
root_cause: "datastore_configuration — <specific_cause>"
evidence:
  - type: configuration_issue
    content: "<details>"
severity: MEDIUM
mitigation:
  immediate: "Fix configuration issue"
  long_term: "Implement configuration monitoring"
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

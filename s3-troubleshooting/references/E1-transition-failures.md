---
title: "E1 — Lifecycle Transition Failures"
description: "Diagnose S3 lifecycle transition failures including minimum duration and size requirements"
status: active
severity: MEDIUM
triggers:
  - "Lifecycle transition not working"
  - "Objects not transitioning"
  - "Storage class not changing"
  - "Lifecycle rule"
owner: devops-agent
objective: "Identify why lifecycle transitions are not occurring and fix the rule configuration"
context: "Lifecycle transitions have minimum duration requirements: 30 days for Standard-IA/One Zone-IA/Intelligent-Tiering, 90 days for Glacier Instant Retrieval, 90 days for Glacier Flexible Retrieval, 180 days for Glacier Deep Archive. Objects smaller than 128 KB are NOT transitioned to IA or Intelligent-Tiering. Waterfall transitions must respect minimum durations between each tier."
---

## Phase 1 — Triage

MUST:
- Check lifecycle configuration: `aws s3api get-bucket-lifecycle-configuration --bucket <bucket>`
- Verify the rule status is Enabled
- Check the filter (prefix, tags) matches the target objects
- Verify transition days meet minimum requirements for the target storage class
- Check object sizes — objects < 128 KB are not transitioned to IA/Intelligent-Tiering

SHOULD:
- Check if multiple rules conflict or overlap on the same objects
- Verify the transition waterfall order is correct (Standard → IA → Glacier → Deep Archive)
- Check CloudWatch metrics for lifecycle operations

MAY:
- Use S3 Inventory to verify object storage classes
- Check if objects were uploaded recently (transitions are based on creation date)

## Phase 2 — Remediate

MUST:
- Fix transition days to meet minimums (30 days for IA, 90 days for Glacier, 180 days for Deep Archive)
- Ensure waterfall transitions have correct gaps (e.g., IA at 30 days, Glacier at 90 days minimum)
- Verify the rule filter matches the intended objects

SHOULD:
- Combine overlapping rules to avoid confusion
- Use S3 Intelligent-Tiering for automatic optimization without lifecycle rules
- Add lifecycle rule for aborting incomplete multipart uploads

MAY:
- Use S3 Storage Lens to analyze storage class distribution
- Set up S3 Inventory to track transition progress

## Common Issues

- symptoms: "Objects remain in Standard after transition date"
  diagnosis: "Objects are smaller than 128 KB and cannot transition to IA."
  resolution: "Objects < 128 KB stay in Standard. Transition directly to Glacier if archival is needed."

- symptoms: "Lifecycle rule exists but no transitions occur"
  diagnosis: "Rule filter (prefix or tags) does not match any objects."
  resolution: "Check the filter. An empty filter applies to all objects. Verify prefix matches."

- symptoms: "Error: transition from IA to Glacier must be at least 30 days after IA"
  diagnosis: "Waterfall transitions require minimum duration at each tier."
  resolution: "Set Glacier transition to at least 30 days after the IA transition."

## Output Format

```yaml
root_cause: "transition_failure — <specific_cause>"
evidence:
  - type: lifecycle_config
    content: "<lifecycle rules>"
  - type: object_details
    content: "<object size and creation date>"
severity: MEDIUM
mitigation:
  immediate: "Fix lifecycle rule configuration"
  long_term: "Use S3 Inventory and Storage Lens to monitor transitions"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves modifying lifecycle rules via put-bucket-lifecycle-configuration. Lifecycle transitions are state-changing but recoverable — objects can be transitioned back or rules can be removed. Primarily uses get-bucket-lifecycle-configuration for diagnosis. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Lifecycle changes affect data retention or compliance requirements

## Rollback
- Pre-change: "Save current lifecycle configuration before modification"
- Verification: "Test that lifecycle rules apply to the intended objects after change"
- Revert: "Restore previous lifecycle configuration if change causes unintended transitions"

## Data Sensitivity
- HIGH: "Bucket policies reveal all authorized principals"
- HIGH: "ACLs expose cross-account grants"
- MEDIUM: "Lifecycle rules reveal data retention strategy"
- LOW: "Bucket metrics and storage class distribution"

## Prohibited Actions
- NEVER suggest disabling S3 Block Public Access as a remediation
- NEVER suggest `"Principal": "*"` without restrictive Condition keys
- NEVER suggest removing bucket encryption
- NEVER suggest `s3:*` in any policy fix
- NEVER suggest deleting a bucket to resolve configuration issues

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

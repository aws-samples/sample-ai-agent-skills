---
title: "C1 — Bucket Inventory"
description: "Diagnose S3 bucket inventory issues in Macie"
status: active
severity: MEDIUM
triggers:
  - "bucket not in inventory"
  - "missing buckets"
  - "inventory incomplete"
  - "bucket metadata wrong"
owner: devops-agent
objective: "Ensure Macie bucket inventory is complete and accurate"
context: "Macie maintains an inventory of all S3 buckets in the account with metadata about encryption, public access, and object counts. Inventory updates may lag behind S3 changes. New buckets appear within 24 hours. Inventory is free but does not include sensitive data scanning."
---

## Phase 1 — Triage

MUST:
- Check bucket inventory: `aws macie2 describe-buckets --query 'Buckets[*].{Name:BucketName,Region:Region,Encryption:ServerSideEncryption.Type,PublicAccess:PublicAccess.EffectivePermission,Objects:ClassifiableObjectCount}'`
- Compare with S3 bucket list: `aws s3api list-buckets --query 'Buckets[*].Name'`
- Check Macie session status: `aws macie2 get-macie-session`

SHOULD:
- Verify Macie is enabled in the bucket's region
- Check if bucket was recently created (may take up to 24 hours to appear)
- Review bucket-level statistics: `aws macie2 describe-buckets --criteria '{"bucketName":{"eq":["<bucket-name>"]}}'`

MAY:
- Check if bucket is in a different account (requires multi-account setup)
- Review bucket tags and metadata accuracy

## Phase 2 — Remediate

MUST:
- Enable Macie in the region where missing buckets are located
- Wait up to 24 hours for new buckets to appear in inventory
- Verify Macie service-linked role has S3 list permissions

SHOULD:
- Set up multi-account Macie for cross-account bucket visibility
- Review inventory regularly for accuracy

MAY:
- Use describe-buckets with criteria to filter specific buckets
- Export inventory for compliance reporting

## Common Issues

- symptoms: "Bucket not appearing in Macie inventory"
  diagnosis: "Macie not enabled in the bucket's region, or bucket is new."
  resolution: "Enable Macie in the region. Wait 24 hours for new buckets."

- symptoms: "Bucket metadata shows wrong encryption status"
  diagnosis: "Inventory may lag behind recent S3 configuration changes."
  resolution: "Wait for inventory refresh. Check S3 directly for current status."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable Macie in missing regions | YELLOW | Enables a new service with cost implications |
| Wait for inventory refresh | GREEN | No action required |
| Set up multi-account Macie | YELLOW | Organization-wide service enablement |

## Escalation Conditions

- Compliance-critical buckets missing from inventory — escalate to compliance team
- Macie not enabled in required regions — escalate to security team
- Inventory metadata inaccurate for audit purposes — escalate to audit team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws macie2 describe-buckets` | HIGH | Exposes bucket names, encryption, public access, and object counts |
| `aws s3api list-buckets` | MEDIUM | Exposes all bucket names |
| `aws macie2 get-macie-session` | LOW | Session status only |

## Prohibited Actions

- **NEVER** suggest disabling Macie as a troubleshooting step
- **NEVER** suggest ignoring sensitive data findings without review
- **NEVER** rely solely on Macie inventory for security decisions without cross-referencing S3

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled Macie in new region | Disable Macie in that region: `aws macie2 disable-macie` |
| Set up multi-account Macie | Remove member associations |

## Output Format

```yaml
root_cause: "bucket_inventory — <specific_cause>"
evidence:
  - type: inventory
    content: "<bucket inventory status>"
  - type: s3_buckets
    content: "<actual S3 bucket list>"
severity: MEDIUM
mitigation:
  immediate: "Enable Macie in missing regions"
  long_term: "Monitor inventory completeness regularly"
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
  - "NEVER suggest disabling Macie to reduce costs without review"
  - "NEVER suggest ignoring sensitive data findings"
  - "NEVER suggest removing S3 bucket monitoring"

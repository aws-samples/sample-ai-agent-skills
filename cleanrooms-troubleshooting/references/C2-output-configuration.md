---
title: "C2 — Output Configuration Issues"
description: "Diagnose Clean Rooms query output problems including S3 destination and result delivery"
status: active
severity: MEDIUM
triggers:
  - "query output"
  - "results not appearing"
  - "output s3"
  - "result configuration"
  - "output bucket"
owner: devops-agent
objective: "Resolve query output configuration issues and ensure results are delivered"
context: "Query results are delivered to an S3 bucket in the receiving member's account. The member must have CAN_RECEIVE_RESULTS ability and a properly configured output location. Common issues include wrong S3 bucket, missing permissions, and output format errors."
---

## Phase 1 — Triage

MUST:
- Check membership result configuration: `aws cleanrooms get-membership --membership-identifier <id>`
- Verify the member has CAN_RECEIVE_RESULTS ability
- Check S3 bucket exists and is in the same account: `aws s3api head-bucket --bucket <bucket>`
- Verify the Clean Rooms service role has S3 write permissions

SHOULD:
- Check S3 bucket policy allows Clean Rooms to write
- Verify KMS key permissions if bucket uses KMS encryption
- Check query status for output-related errors

MAY:
- List objects in the output prefix: `aws s3 ls s3://<bucket>/<prefix>/`
- Check CloudTrail for S3 write events from Clean Rooms

## Phase 2 — Remediate

MUST:
- Configure result settings on membership:
  ```
  aws cleanrooms update-membership --membership-identifier <id> --default-result-configuration '{"outputConfiguration":{"s3":{"bucket":"<bucket>","keyPrefix":"<prefix>","resultFormat":"CSV"}},"roleArn":"<service-role-arn>"}'
  ```
- Ensure service role has `s3:PutObject` on the output bucket
- If KMS encrypted, grant `kms:GenerateDataKey` to the service role

SHOULD:
- Use a dedicated output bucket for Clean Rooms results
- Configure appropriate result format (CSV or Parquet)

## Output Format

```yaml
root_cause: "output_configuration — <specific_cause>"
evidence:
  - type: membership_config
    content: "<membership result configuration>"
  - type: s3_access
    content: "<S3 bucket access status>"
severity: MEDIUM
mitigation:
  immediate: "Fix output configuration or S3 permissions"
  long_term: "Standardize output configuration across collaborations"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Configure result settings on membership | YELLOW |
| Ensure service role has s3:PutObject | YELLOW |
| Grant kms:GenerateDataKey to service role | YELLOW |
| Use a dedicated output bucket | GREEN |
| Configure appropriate result format | GREEN |

## Escalation Conditions

- Fix requires modifying production output destinations
- Fix involves changing S3 bucket permissions or KMS key access

## Data Sensitivity

- **HIGH**: output configuration determines where query results are delivered
- S3 bucket and KMS key configurations control access to collaboration results

## Prohibited Actions

- NEVER suggest disabling privacy controls
- NEVER suggest exporting raw data from collaborations

## Phase 3 — Rollback

1. If output configuration was changed, restore previous result settings on membership
2. If S3 permissions were modified, revert bucket policy and IAM role
3. If KMS key access was granted, revoke if no longer needed
4. Verify rollback by confirming `aws cleanrooms get-membership` shows expected output configuration

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"

---
title: "D3 — Replication Lag / RTC"
description: "Diagnose S3 replication latency and Replication Time Control issues"
status: active
severity: MEDIUM
triggers:
  - "Replication lag"
  - "Replication slow"
  - "Replication Time Control"
  - "Objects pending replication"
owner: devops-agent
objective: "Identify causes of replication lag and optimize replication performance"
context: "S3 replication is asynchronous. Most objects replicate within 15 minutes, but large objects or high throughput can cause lag. S3 Replication Time Control (RTC) provides an SLA of 99.99% of objects replicated within 15 minutes. RTC requires replication metrics to be enabled."
---

## Phase 1 — Triage

MUST:
- Check replication metrics: `aws cloudwatch get-metric-statistics --namespace AWS/S3 --metric-name ReplicationLatency --dimensions Name=SourceBucket,Value=<bucket> Name=DestinationBucket,Value=<dest-bucket> Name=RuleId,Value=<rule-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check pending operations: `aws cloudwatch get-metric-statistics --namespace AWS/S3 --metric-name OperationsPendingReplication --dimensions Name=SourceBucket,Value=<bucket> Name=DestinationBucket,Value=<dest-bucket> Name=RuleId,Value=<rule-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Verify replication metrics are enabled in the replication rule
- Check if RTC is enabled: `aws s3api get-bucket-replication --bucket <bucket> --query 'ReplicationConfiguration.Rules[].ExistingObjectReplication'`

SHOULD:
- Check object sizes — large objects take longer to replicate
- Verify no throttling on source or destination (503 SlowDown)
- Check if KMS encryption is adding latency (KMS API rate limits)

MAY:
- Check S3 request metrics for the source bucket
- Review if multipart upload objects are causing delays

## Phase 2 — Remediate

MUST:
- Enable replication metrics if not already enabled
- If SLA is required: enable S3 Replication Time Control (RTC)
- Monitor OperationsPendingReplication and ReplicationLatency metrics

SHOULD:
- Set up CloudWatch alarms for replication lag thresholds
- For large objects: ensure multipart upload is used (replication handles parts)
- If KMS throttling: request KMS quota increase

MAY:
- Enable S3 Replication Notifications (EventBridge) for failed replication events
- Consider S3 Batch Replication for backfilling objects that failed to replicate

## Common Issues

- symptoms: "Replication latency spikes during bulk uploads"
  diagnosis: "High volume of objects overwhelms replication throughput."
  resolution: "This is expected. Enable RTC for SLA guarantees. Monitor OperationsPendingReplication."

- symptoms: "Large objects take hours to replicate"
  diagnosis: "Objects > 100 MB use multipart replication which takes longer."
  resolution: "This is expected behavior. RTC provides 15-minute SLA for 99.99% of objects."

- symptoms: "Replication metrics show 0 but objects are not in destination"
  diagnosis: "Replication metrics are not enabled on the rule."
  resolution: "Enable metrics in the replication rule configuration."

## Output Format

```yaml
root_cause: "replication_lag — <specific_cause>"
evidence:
  - type: replication_metrics
    content: "<latency and pending operations>"
  - type: object_sizes
    content: "<size distribution>"
severity: MEDIUM
mitigation:
  immediate: "Enable replication metrics and RTC if SLA required"
  long_term: "Set up CloudWatch alarms and EventBridge notifications"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves enabling Replication Time Control and modifying replication rules via put-bucket-replication. Changes to replication configuration are state-changing but recoverable. Primarily uses get-bucket-replication and CloudWatch metrics for diagnosis. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Replication SLA changes affect compliance or data residency requirements

## Rollback
- Pre-change: "Save current replication configuration before modification"
- Verification: "Test replication with a new object upload after change"
- Revert: "Restore previous replication configuration if change causes unintended replication behavior"

## Data Sensitivity
- HIGH: "Bucket policies reveal all authorized principals"
- HIGH: "ACLs expose cross-account grants"
- MEDIUM: "Replication configuration reveals destination buckets and cross-account relationships"
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

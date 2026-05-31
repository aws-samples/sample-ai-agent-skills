---
title: "H1 — Event Notification Failures"
description: "Diagnose S3 event notification delivery failures to SNS, SQS, Lambda, and EventBridge"
status: active
severity: HIGH
triggers:
  - "Event notification not working"
  - "S3 notification failed"
  - "Events not triggering"
  - "NotificationConfiguration"
owner: devops-agent
objective: "Identify why S3 event notifications are not being delivered and fix the configuration"
context: "S3 event notifications can target SNS, SQS, Lambda, or EventBridge. The destination must have a resource policy allowing S3 to publish. Each event type can only have ONE destination per notification configuration (except EventBridge which receives all events). Overlapping event configurations cause errors."
---

## Phase 1 — Triage

MUST:
- Check notification configuration: `aws s3api get-bucket-notification-configuration --bucket <bucket>`
- Verify the destination exists and is in the same region
- Check the destination resource policy allows s3.amazonaws.com to publish
- Verify no overlapping event types in the configuration

SHOULD:
- Check CloudTrail for PutBucketNotificationConfiguration errors
- Verify the event type matches the operation (s3:ObjectCreated:* vs s3:ObjectCreated:Put)
- Check if the prefix/suffix filter matches the target objects

MAY:
- Test with a manual upload and check the destination for the event
- Check destination-side logs (Lambda logs, SQS messages, SNS delivery logs)

## Phase 2 — Remediate

MUST:
- Fix the destination resource policy to allow S3:
  - SQS: `"aws:SourceArn": "arn:aws:s3:::<bucket>"` in the queue policy
  - SNS: `"aws:SourceArn": "arn:aws:s3:::<bucket>"` in the topic policy
  - Lambda: add permission: `aws lambda add-permission --function-name <fn> --statement-id s3-trigger --action lambda:InvokeFunction --principal s3.amazonaws.com --source-arn arn:aws:s3:::<bucket>`
- Remove overlapping event type configurations
- Ensure the destination is in the same region as the bucket

SHOULD:
- Use EventBridge for complex routing (multiple destinations per event type)
- Add error handling in Lambda functions for malformed events
- Test the notification with a sample upload after fixing

MAY:
- Enable S3 server access logging to verify events are being generated
- Set up dead-letter queues for failed Lambda invocations

## Common Issues

- symptoms: "Notification configuration save fails"
  diagnosis: "Overlapping event types or destination resource policy does not allow S3."
  resolution: "Remove duplicate event types. Update destination policy to allow s3.amazonaws.com."

- symptoms: "Events work for some objects but not others"
  diagnosis: "Prefix or suffix filter in the notification does not match all target objects."
  resolution: "Check the filter rules. Prefix and suffix are case-sensitive."

- symptoms: "Events trigger but destination receives nothing"
  diagnosis: "Destination resource policy blocks S3 or destination is in a different region."
  resolution: "Update the resource policy and ensure same-region deployment."

## Output Format

```yaml
root_cause: "event_notification — <specific_cause>"
evidence:
  - type: notification_config
    content: "<notification configuration>"
  - type: destination_policy
    content: "<resource policy>"
severity: HIGH
mitigation:
  immediate: "Fix destination permissions and event configuration"
  long_term: "Use EventBridge for flexible routing and add monitoring"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves modifying destination resource policies (SQS, SNS, Lambda) and S3 notification configuration via put-bucket-notification-configuration. Resource policy changes are state-changing but recoverable. Uses get-bucket-notification-configuration for diagnosis. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Notification changes affect event-driven production workflows

## Rollback
- Pre-change: "Save current bucket policy/ACL/CORS before modification"
- Verification: "Test access with the specific operation after change"
- Revert: "Restore previous configuration if change causes unintended access"

## Data Sensitivity
- HIGH: "Bucket policies reveal all authorized principals"
- HIGH: "ACLs expose cross-account grants"
- MEDIUM: "Notification configuration reveals integration architecture"
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

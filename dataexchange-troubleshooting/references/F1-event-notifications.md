---
title: "F1 — Event Notifications"
description: "Diagnose Data Exchange event action and notification issues"
status: active
severity: MEDIUM
triggers: ["event action not triggering", "automatic publishing failed", "S3 event not detected", "revision not auto-published"]
owner: devops-agent
objective: "Resolve event action issues including S3 event configuration, automatic revision publishing, and notification delivery"
context: "Event actions automate Data Exchange workflows. The primary use case is automatic revision publishing when new S3 objects are created. Requires correct S3 event notification configuration."
---

## Phase 1 — Triage

MUST:
- List event actions: `aws dataexchange list-event-actions`
- Check event action details: `aws dataexchange get-event-action --event-action-id <id>`
- Verify S3 bucket event notification configuration: `aws s3api get-bucket-notification-configuration --bucket <bucket-name>`

SHOULD:
- Check if S3 events are being generated for new objects
- Verify the event action's data set and revision configuration
- Review CloudTrail for event action execution

MAY:
- Check S3 bucket policy for Data Exchange service access
- Verify Lambda function configuration if using custom event processing

## Phase 2 — Remediate

MUST:
- Configure S3 event notifications to trigger Data Exchange event actions
- Fix event action configuration (correct data set ID, asset source)
- Ensure IAM permissions allow Data Exchange to read from S3 bucket

SHOULD:
- Test event action by uploading a test object to S3
- Verify revision is created and finalized after event triggers

MAY:
- Set up CloudWatch alarms for event action failures
- Configure dead-letter queue for failed event processing

## Safety Ratings

```
safety_ratings:
  - "dataexchange list-event-actions: GREEN — read-only listing"
  - "s3api get-bucket-notification-configuration: GREEN — read-only check"
  - "dataexchange create-event-action: YELLOW — creates automated publishing workflow"
  - "s3api put-bucket-notification-configuration: YELLOW — modifies S3 event routing"
```

## Escalation Conditions

- Event actions fail silently with no error logs
- S3 event notification configuration conflicts with existing notifications
- Automatic publishing creates incorrect revisions affecting subscribers
- Event action rate limiting prevents timely data publishing

## Data Sensitivity

- `aws dataexchange list-event-actions`: LOW — returns event action metadata
- `aws s3api get-bucket-notification-configuration`: MEDIUM — reveals S3 event routing
- `event action configuration`: MEDIUM — contains S3 bucket and data set details
- `CloudTrail event action events`: LOW — returns execution history

## Prohibited Actions

- NEVER configure event actions that publish unvalidated data automatically
- NEVER modify S3 event notifications without checking for existing configurations
- NEVER delete event actions that are actively publishing data without a replacement
- NEVER configure event actions on S3 buckets containing sensitive data without review

## Phase 3 — Rollback

1. If event action was misconfigured, update or delete it: `aws dataexchange delete-event-action --event-action-id <id>`
2. If S3 notifications were modified, restore previous configuration
3. If incorrect revisions were published, notify subscribers and create corrected revision
4. Verify event action is working correctly after fix
5. Document the event configuration and test procedure

## Common Issues

- symptoms: "Event action not triggering — S3 objects created but no revision published"
  diagnosis: "S3 bucket event notifications not configured to send to Data Exchange."
  resolution: "Configure S3 event notification for s3:ObjectCreated:* targeting Data Exchange."

## Output Format

```yaml
root_cause: "event_notifications — <specific_cause>"
evidence:
  - type: event_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix event action configuration"
  long_term: "Monitor event action execution and set up alerts"
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

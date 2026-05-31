---
title: "D1 — Alert Configuration"
description: "Diagnose and resolve alert configuration issues for Lookout for Metrics"
status: active
severity: MEDIUM
triggers: ["alert not working", "no alert notifications", "alert configuration error", "SNS alert failed"]
owner: devops-agent
objective: "Ensure alerts are properly configured and delivering notifications for detected anomalies"
context: "Lookout for Metrics supports SNS and Lambda alert channels. Alerts are triggered when anomaly groups are detected above the configured severity threshold. Alert failures are typically caused by permission issues or misconfigured targets."
---

## Phase 1 — Triage

MUST:
- List alerts for the detector: `aws lookoutmetrics list-alerts --anomaly-detector-arn <arn>`
- Describe alert configuration: `aws lookoutmetrics describe-alert --alert-arn <arn>`
- Verify the alert target (SNS topic or Lambda function) exists and is accessible

SHOULD:
- Check SNS topic subscription status: `aws sns list-subscriptions-by-topic --topic-arn <arn>`
- Verify anomalies are being detected: `aws lookoutmetrics list-anomaly-group-summaries --anomaly-detector-arn <arn> --sensitivity-threshold 50`
- Check alert severity threshold matches expected anomaly severity levels

MAY:
- Review CloudTrail for alert-related API errors
- Test SNS topic by publishing a test message: `aws sns publish --topic-arn <arn> --message "test"`

## Phase 2 — Remediate

MUST:
- Fix SNS topic permissions — ensure Lookout for Metrics can publish to the topic
- Verify SNS topic policy allows `sns:Publish` from lookoutmetrics.amazonaws.com
- Ensure alert severity threshold is not set higher than detected anomaly severities

SHOULD:
- Add SNS topic policy for Lookout for Metrics service principal
- Verify SNS subscriptions are confirmed (not pending)
- Check alert is associated with the correct detector

MAY:
- Create a new alert if the existing one cannot be fixed
- Set up multiple alert channels for redundancy (SNS + Lambda)

## Safety Ratings

```
safety_ratings:
  - "list-alerts: GREEN — read-only alert enumeration"
  - "describe-alert: GREEN — read-only alert configuration"
  - "CreateAlert: YELLOW — creates new alert resource"
  - "sns publish: YELLOW — sends message to topic subscribers"
```

## Escalation Conditions

- Alert configuration is correct but notifications are not delivered
- SNS topic is in a different account and cross-account permissions cannot be configured
- Alert creation fails with internal service errors
- Multiple alerts fail simultaneously across different detectors

## Data Sensitivity

- `aws lookoutmetrics list-alerts`: LOW — returns alert metadata
- `aws lookoutmetrics describe-alert`: MEDIUM — contains alert target ARNs
- `aws sns list-subscriptions-by-topic`: MEDIUM — reveals subscriber endpoints (email, phone, etc.)
- SNS topic policy: LOW — permission configuration only

## Prohibited Actions

- NEVER delete production alerts without creating replacements first
- NEVER publish test messages to production SNS topics that trigger downstream automation
- NEVER grant `sns:*` permissions — use only `sns:Publish` for the specific topic
- NEVER expose subscriber email addresses or phone numbers from SNS subscriptions

## Phase 3 — Rollback

1. If an alert was deleted in error, recreate it with the same configuration
2. If SNS topic policy was modified, revert to previous policy version
3. If a test message triggered downstream automation, notify affected teams
4. Document all alert configuration changes
5. Verify alert delivery by checking for the next anomaly notification

## Common Issues

- symptoms: "Alert exists but no notifications received"
  diagnosis: "SNS topic policy does not allow Lookout for Metrics to publish, or no confirmed subscriptions."
  resolution: "Add SNS topic policy: `aws sns set-topic-attributes --topic-arn <arn> --attribute-name Policy --attribute-value '<policy>'` allowing lookoutmetrics.amazonaws.com."

- symptoms: "CreateAlert returns ValidationException"
  diagnosis: "Invalid alert target ARN, unsupported alert channel type, or detector not found."
  resolution: "Verify the SNS topic or Lambda function ARN is correct and the detector ARN exists."

## Output Format

```yaml
root_cause: "alert_configuration — <specific_cause>"
evidence:
  - type: alert_error
    content: "<alert configuration and error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix alert configuration or permissions"
  long_term: "Implement alert monitoring and delivery verification"
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

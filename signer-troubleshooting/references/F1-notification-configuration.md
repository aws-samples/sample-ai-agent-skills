---
title: "F1 — Notification Configuration"
description: "Diagnose AWS Signer notification configuration and delivery issues"
status: active
severity: LOW
triggers: ["signing notifications not working", "SNS notification failed", "no signing alerts"]
owner: devops-agent
objective: "Resolve Signer notification configuration and delivery issues"
context: "AWS Signer can send notifications via SNS for signing job status changes. The SNS topic must have a resource policy allowing Signer to publish. Notifications cover job completion (Succeeded/Failed)."
---

## Phase 1 — Triage

MUST:
- Check if SNS topic exists: `aws sns get-topic-attributes --topic-arn <topic-arn>`
- Verify SNS topic policy allows Signer to publish
- Check SNS subscriptions: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`

SHOULD:
- Verify the signing job was configured with the notification ARN
- Check SNS delivery logs for failed deliveries

MAY:
- Test SNS topic by publishing a test message
- Review CloudTrail for SNS publish failures from Signer

## Phase 2 — Remediate

MUST:
- Create SNS topic if missing: `aws sns create-topic --name signer-notifications`
- Add resource policy allowing Signer: `aws sns set-topic-attributes --topic-arn <arn> --attribute-name Policy --attribute-value '{"Statement":[{"Effect":"Allow","Principal":{"Service":"signer.amazonaws.com"},"Action":"SNS:Publish","Resource":"<topic-arn>"}]}'`
- Subscribe to the topic: `aws sns subscribe --topic-arn <arn> --protocol email --notification-endpoint <email>`

SHOULD:
- Confirm subscription (check email for confirmation link)
- Test notification delivery with a signing job

MAY:
- Set up SNS to trigger Lambda for automated responses to signing events
- Configure SNS message filtering for specific job statuses

## Common Issues

- symptoms: "Signing jobs complete but no notifications received"
  diagnosis: "SNS topic policy missing Signer service permission or subscription not confirmed."
  resolution: "Add signer.amazonaws.com to SNS topic policy. Confirm the subscription."

- symptoms: "SNS topic exists but Signer cannot publish"
  diagnosis: "SNS topic resource policy does not allow signer.amazonaws.com."
  resolution: "Update SNS topic policy to allow Publish from signer.amazonaws.com."

## Safety Ratings

```
safety_ratings:
  - "sns get-topic-attributes: GREEN — read-only topic details"
  - "sns list-subscriptions-by-topic: GREEN — read-only subscription listing"
  - "sns create-topic: YELLOW — creates new SNS topic"
  - "sns set-topic-attributes: YELLOW — modifies topic access policy"
```

## Escalation Conditions

- Notifications required for compliance but SNS delivery consistently fails
- SNS topic in a different account requires cross-account publish permissions
- High volume of signing jobs causing SNS throttling
- Notification delivery latency exceeds acceptable thresholds

## Data Sensitivity

- `aws sns get-topic-attributes`: MEDIUM — reveals topic policy and configuration
- SNS subscriptions: MEDIUM — reveal notification recipients (email, Lambda, etc.)
- Signing job notifications: LOW — contain job status and metadata
- SNS topic policy: MEDIUM — reveals service access permissions

## Prohibited Actions

- NEVER grant SNS:* permissions to the Signer service — only SNS:Publish
- NEVER subscribe untrusted endpoints to signing notification topics
- NEVER delete SNS topics that other services depend on
- NEVER store sensitive data in SNS notification messages

## Phase 3 — Rollback

1. If SNS topic policy was modified incorrectly, restore previous policy
2. If unwanted subscriptions were created, unsubscribe them
3. If a topic was created incorrectly, delete it: `aws sns delete-topic --topic-arn <arn>`
4. Verify notification delivery is restored after changes
5. Document the notification configuration changes

## Output Format

```yaml
root_cause: "notification_configuration — <specific_cause>"
evidence:
  - type: notification_error
    content: "<error details>"
severity: LOW
mitigation:
  immediate: "Fix notification configuration"
  long_term: "Automate notification setup via IaC"
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

---
title: "D2 — SNS Integration"
description: "Diagnose and resolve AMP alert manager SNS notification failures"
status: active
severity: HIGH
triggers:
  - "SNS"
  - "alert notification"
  - "not receiving alerts"
  - "sns_configs"
  - "topic"
  - "notification"
owner: devops-agent
objective: "Identify and fix SNS integration issues for AMP alerting"
context: "AMP alert manager uses Amazon SNS as the notification receiver. Alert notifications are sent to SNS topics, which then fan out to subscriptions (email, Lambda, Slack via Chatbot, etc.). AMP does NOT support direct Slack, email, or PagerDuty receivers — only SNS."
---

## Phase 1 — Triage

MUST:
- Check alert manager definition: `aws amp describe-alert-manager-definition --workspace-id <workspace-id>`
- Verify SNS topic exists: `aws sns get-topic-attributes --topic-arn <topic-arn>`
- Check SNS topic subscriptions: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`
- Verify IAM permissions for AMP to publish to SNS

SHOULD:
- Check if alerting rules are actually firing (test the alert expression as a query)
- Verify the alert manager YAML has correct sns_configs with valid topic_arn
- Check SNS delivery logs for failed deliveries
- Verify the SNS topic is in the same region as the AMP workspace

MAY:
- Test SNS topic directly: `aws sns publish --topic-arn <topic-arn> --message "test"`
- Check CloudTrail for SNS Publish failures from AMP
- Review SNS topic access policy

## Phase 2 — Remediate

MUST:
- If alert manager not configured: create definition:
  ```
  aws amp create-alert-manager-definition --workspace-id <workspace-id> \
    --data file://alertmanager.yaml
  ```
- If SNS topic missing: create it and update alert manager definition
- If IAM permissions missing: add SNS publish permissions to AMP's service role
- Ensure alert manager YAML uses sns_configs (not slack_configs, email_configs, etc.)

SHOULD:
- Configure alert manager with proper routing and grouping:
  ```yaml
  alertmanager_config: |
    route:
      receiver: default
      group_by: ['alertname']
    receivers:
      - name: default
        sns_configs:
          - topic_arn: arn:aws:sns:<region>:<account>:<topic>
            sigv4:
              region: <region>
            message: |
              {{ range .Alerts }}
              Alert: {{ .Labels.alertname }}
              {{ end }}
  ```
- Set up SNS subscriptions for downstream notification channels
- Configure alert grouping to avoid notification storms

MAY:
- Use SNS message filtering for routing different alerts to different subscribers
- Set up dead-letter queue for failed SNS deliveries
- Implement Lambda function for custom alert formatting

## Common Issues

- symptoms: "Alerts firing but no SNS notifications received"
  diagnosis: "AMP cannot publish to SNS topic — IAM permission or topic policy issue."
  resolution: "Verify AMP has sns:Publish permission on the topic. Check SNS topic access policy allows AMP service."

- symptoms: "Alert manager definition upload fails with ValidationException"
  diagnosis: "Alert manager YAML has invalid syntax or unsupported receiver type."
  resolution: "AMP only supports sns_configs receivers. Remove any slack_configs, email_configs, etc. Validate YAML syntax."

- symptoms: "SNS notifications received but content is empty or malformed"
  diagnosis: "Alert manager template is incorrect or missing."
  resolution: "Update the message template in sns_configs. Use Go template syntax for alert labels and annotations."

- symptoms: "Alerts are delayed significantly"
  diagnosis: "Alert group_wait, group_interval, or repeat_interval is too long."
  resolution: "Adjust timing parameters: group_wait (default 30s), group_interval (default 5m), repeat_interval (default 4h)."

## Safety Ratings

```
safety_ratings:
  - "describe-alert-manager-definition: GREEN — read-only config check"
  - "sns get-topic-attributes: GREEN — read-only SNS check"
  - "sns list-subscriptions-by-topic: GREEN — read-only subscription check"
  - "create-alert-manager-definition: YELLOW — creates alerting configuration"
  - "put-alert-manager-definition: YELLOW — updates alerting configuration"
```

## Escalation Conditions

- Production alerts not reaching on-call teams via SNS
- SNS topic permissions require cross-account or cross-team coordination
- Alert manager definition upload fails repeatedly
- Alert notification delays exceed acceptable thresholds

## Data Sensitivity

- Alert manager configuration: MEDIUM — reveals alerting logic and SNS topic ARNs
- SNS topic attributes: MEDIUM — reveals topic configuration and access policies
- SNS subscriptions: MEDIUM — reveals who receives alert notifications

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER use slack_configs, email_configs, or other non-SNS receivers (only sns_configs supported)
- NEVER delete the alert manager definition without backing it up first

## Phase 3 — Rollback

1. If alert manager definition was changed incorrectly, restore from backup: `aws amp put-alert-manager-definition --workspace-id <id> --data file://previous-alertmanager.yaml`
2. If SNS topic permissions were changed, restore original topic access policy
3. If alert manager definition was deleted, recreate from backup: `aws amp create-alert-manager-definition --workspace-id <id> --data file://alertmanager.yaml`
4. Verify alerts fire and notifications are delivered end-to-end after rollback
5. Document the configuration change and rollback for audit

## Output Format

```yaml
root_cause: "SNS integration — <specific_cause>"
evidence:
  - type: alertmanager_config
    content: "<alert manager definition status>"
  - type: sns_topic
    content: "<SNS topic attributes and subscriptions>"
severity: HIGH
mitigation:
  immediate: "Fix alert manager configuration or SNS permissions"
  long_term: "Implement end-to-end alert testing, monitor SNS delivery metrics"
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

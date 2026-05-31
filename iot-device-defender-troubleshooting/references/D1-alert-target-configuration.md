---
title: "D1 — Alert Target Configuration"
description: "Diagnose alert target configuration issues for security profiles"
status: active
severity: HIGH
triggers: ["no violation alerts", "alert target error", "SNS not receiving alerts", "notification not configured"]
owner: devops-agent
objective: "Resolve alert target configuration issues including SNS topic setup, IAM permissions, and per-profile configuration"
context: "Alert targets are configured per security profile. They define where violation notifications are sent (SNS topics). Without alert targets, violations are recorded but no notifications are delivered."
---

## Phase 1 — Triage

MUST:
- Check security profile alert targets: `aws iot describe-security-profile --security-profile-name <name> --query 'alertTargets'`
- Verify SNS topic exists: `aws sns get-topic-attributes --topic-arn <topic-arn>`
- Check SNS topic policy allows IoT service: `aws sns get-topic-attributes --topic-arn <topic-arn> --query 'Attributes.Policy'`

SHOULD:
- Verify IAM role for alert target has sns:Publish permission
- Check SNS subscription status for the topic
- Test SNS topic by publishing a test message

MAY:
- Check CloudWatch metrics for SNS delivery failures
- Review CloudTrail for Publish errors from IoT service

## Phase 2 — Remediate

MUST:
- Configure alert target on security profile: `aws iot update-security-profile --security-profile-name <name> --alert-targets '{"SNS":{"alertTargetArn":"<topic-arn>","roleArn":"<role-arn>"}}'`
- Add IoT service principal to SNS topic policy
- Ensure IAM role allows sns:Publish to the target topic

SHOULD:
- Create SNS subscriptions (email, Lambda, SQS) for the alert topic
- Verify end-to-end alert delivery by triggering a test violation

MAY:
- Configure separate alert targets for different security profiles
- Set up SNS message filtering for violation severity

## Safety Ratings

```
safety_ratings:
  - "iot describe-security-profile: GREEN — read-only profile check"
  - "sns get-topic-attributes: GREEN — read-only topic check"
  - "iot update-security-profile (alert targets): YELLOW — modifies notification routing"
  - "sns set-topic-attributes (policy): YELLOW — modifies topic access control"
```

## Escalation Conditions

- Alert targets configured but no notifications delivered for active violations
- SNS topic policy cannot be modified due to organizational restrictions
- IAM role for alert target is compromised or deleted
- Alert delivery latency exceeds acceptable thresholds

## Data Sensitivity

- `aws iot describe-security-profile alertTargets`: MEDIUM — contains SNS topic ARN and IAM role
- `aws sns get-topic-attributes`: MEDIUM — contains topic policy and configuration
- `violation notifications`: MEDIUM — contain device identifiers and violation details
- `IAM role for alerts`: MEDIUM — role ARN and permissions

## Prohibited Actions

- NEVER configure alert targets pointing to public SNS topics
- NEVER grant overly broad SNS publish permissions to the IoT service
- NEVER remove alert targets from security profiles monitoring critical devices
- NEVER ignore missing alert target configuration — violations will go unnoticed

## Phase 3 — Rollback

1. If alert target was misconfigured, update security profile with correct SNS topic ARN
2. If SNS topic policy was modified incorrectly, restore previous policy
3. If IAM role was changed, restore previous permissions
4. Verify alert delivery by checking SNS delivery metrics
5. Document the alert configuration and test procedure

## Common Issues

- symptoms: "No violation notifications despite active violations"
  diagnosis: "Alert target not configured on the security profile."
  resolution: "Add alert target with SNS topic ARN and IAM role to the security profile."

## Output Format

```yaml
root_cause: "alert_target — <specific_cause>"
evidence:
  - type: alert_config_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Configure alert target"
  long_term: "Monitor alert delivery and set up dead-letter queue"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"

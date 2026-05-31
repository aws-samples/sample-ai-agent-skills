---
title: "D2 — Compliance Logging"
description: "Diagnose compliance logging and audit issues"
status: active
severity: HIGH
triggers: ["compliance logs missing", "audit log error", "logging not working", "compliance data gap"]
owner: devops-agent
objective: "Resolve compliance logging configuration and data collection issues"
context: "Compliance logging captures message metadata (not content) for audit purposes. Requires CloudWatch Logs or S3 configuration."
---

## Phase 1 — Triage

MUST:
- Verify network compliance settings: `aws wickr get-network --network-id <id>`
- Check CloudWatch log group exists: `aws logs describe-log-groups --log-group-name-prefix /aws/wickr/`
- Review recent compliance log entries: `aws logs filter-log-events --log-group-name /aws/wickr/<network-id> --start-time <epoch-ms> --limit 20`

SHOULD:
- Verify IAM role for compliance logging has correct permissions
- Check CloudWatch Logs retention settings
- Review CloudTrail for compliance configuration changes

MAY:
- Check for gaps in compliance log timestamps
- Verify log format matches expected schema

## Phase 2 — Remediate

MUST:
- Create CloudWatch log group if missing: `aws logs create-log-group --log-group-name /aws/wickr/<network-id>`
- Fix IAM role permissions for log delivery
- Enable compliance logging in the Wickr admin console

SHOULD:
- Set appropriate log retention period
- Configure log metric filters for compliance monitoring
- Set up CloudWatch alarms for logging failures

MAY:
- Export compliance logs to S3 for long-term archival
- Configure log subscriptions for real-time analysis

## Safety Ratings

```
safety_ratings:
  - "wickr get-network: GREEN — read-only network details"
  - "logs describe-log-groups: GREEN — read-only log group listing"
  - "logs filter-log-events: GREEN — read-only log retrieval"
  - "logs create-log-group: YELLOW — creates new log group resource"
```

## Escalation Conditions

- Compliance logging gap exceeds regulatory retention requirements
- Log data appears corrupted or incomplete
- Compliance logging cannot be enabled due to service issues
- Audit team requires immediate access to missing compliance data

## Data Sensitivity

- `aws wickr get-network`: LOW — returns network metadata
- Compliance logs: MEDIUM — contain message metadata (sender, recipient, timestamps)
- Compliance log content: MEDIUM — does NOT contain message text (E2E encrypted)
- Log group configuration: LOW — logging infrastructure metadata

## Prohibited Actions

- NEVER delete compliance logs without regulatory approval
- NEVER claim compliance logs contain message content
- NEVER reduce log retention below regulatory requirements
- NEVER share compliance logs externally without legal review

## Phase 3 — Rollback

1. If log group was deleted, recreate it with original configuration
2. If retention settings were changed, restore original retention period
3. If compliance logging was disabled, re-enable immediately
4. Verify compliance logs are flowing after rollback
5. Document the compliance logging gap for audit records

## Common Issues

- symptoms: "No compliance logs appearing in CloudWatch"
  diagnosis: "Compliance logging not enabled or IAM role lacks log delivery permissions."
  resolution: "Enable compliance logging in admin console. Verify IAM role has logs:CreateLogStream and logs:PutLogEvents."

- symptoms: "Compliance logs have gaps in timestamps"
  diagnosis: "Logging service experienced temporary delivery failure."
  resolution: "Check CloudTrail for service events during the gap period. Verify IAM permissions are intact."

## Output Format

```yaml
root_cause: "compliance_logging — <specific_cause>"
evidence:
  - type: logging_status
    content: "<log details>"
severity: HIGH
mitigation:
  immediate: "Restore compliance logging"
  long_term: "Implement logging monitoring and gap detection"
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

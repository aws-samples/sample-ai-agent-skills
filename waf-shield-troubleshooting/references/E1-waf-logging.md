---
title: "E1 — WAF Logging Setup Issues"
description: "Diagnose WAF logging configuration problems and missing logs"
status: active
severity: HIGH
triggers:
  - "WAF logging"
  - "WAF logs"
  - "no WAF logs"
  - "logging not working"
  - "aws-waf-logs"
owner: devops-agent
objective: "Identify and fix WAF logging configuration issues to ensure complete request visibility"
context: "WAF logs can be sent to Amazon Kinesis Data Firehose, S3, or CloudWatch Logs. The destination name MUST start with 'aws-waf-logs-'. Logs include request headers, matched rules, labels, and actions but NOT the request body. Log filtering can reduce volume by logging only blocked or specific rule matches. Redacted fields can be configured to mask sensitive data."
---

## Phase 1 — Triage

MUST:
- Check if logging is configured: `aws wafv2 get-logging-configuration --resource-arn <web-acl-arn>`
- If not configured, check if a logging configuration exists: `aws wafv2 list-logging-configurations --scope <scope>`
- Verify the log destination exists and the name starts with `aws-waf-logs-`
- Check IAM permissions for WAF to write to the destination

SHOULD:
- Check if log filtering is configured (may be excluding the requests you're looking for)
- Verify the Kinesis Firehose delivery stream / S3 bucket / CloudWatch log group is receiving data
- Check for redacted fields that may hide needed information

MAY:
- Check CloudTrail for PutLoggingConfiguration events
- Verify the log destination is in the correct region (same as the web ACL for REGIONAL)

## Phase 2 — Remediate

MUST:
- Create the log destination with the `aws-waf-logs-` prefix
- For Kinesis Firehose: `aws firehose create-delivery-stream --delivery-stream-name aws-waf-logs-<name> --s3-destination-configuration ...`
- Enable logging: `aws wafv2 put-logging-configuration --logging-configuration ResourceArn=<web-acl-arn>,LogDestinationConfigs=<destination-arn>`
- Grant WAF permissions to write to the destination (resource-based policy for S3/CloudWatch, service role for Firehose)

SHOULD:
- Configure log filtering to reduce costs while maintaining visibility on blocked requests
- Set up redacted fields for sensitive headers (Authorization, Cookie)
- Configure log retention policies on the destination

MAY:
- Set up Athena queries for S3-based WAF logs
- Create CloudWatch Insights queries for CloudWatch Logs-based WAF logs
- Implement log analysis automation with Lambda

## Common Issues

- symptoms: "PutLoggingConfiguration fails with InvalidParameterException"
  diagnosis: "Log destination name does not start with 'aws-waf-logs-'."
  resolution: "Rename or recreate the destination with the 'aws-waf-logs-' prefix."

- symptoms: "Logging configured but no logs appearing"
  diagnosis: "IAM permissions are missing. WAF cannot write to the destination."
  resolution: "Add the required resource-based policy or service role permissions."

- symptoms: "Logs missing for some requests"
  diagnosis: "Log filtering is configured to only log specific actions or rules."
  resolution: "Review and adjust the logging filter configuration."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create log destination | YELLOW | New resource; can be deleted |
| Enable logging | YELLOW | Configuration change; reversible |
| Configure log filtering | YELLOW | Filter change; reversible |
| Set up redacted fields | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Production web ACL logging changes
- Shield Advanced configuration changes
- Log destination permission changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-logging-configuration` | LOW | Logging configuration |
| WAF logs | HIGH | Full request headers and client IPs |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting
- NEVER disable WAF logging without explicit approval

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Logging enablement | Disable via `delete-logging-configuration` |
| Log filter change | Revert filter via `put-logging-configuration` |
| Redacted fields change | Revert via `put-logging-configuration` |

## Output Format

```yaml
root_cause: "waf_logging — <specific_cause>"
evidence:
  - type: logging_config
    content: "<current logging configuration>"
  - type: destination
    content: "<log destination ARN and status>"
  - type: permissions
    content: "<IAM permissions status>"
severity: HIGH
mitigation:
  immediate: "Fix logging destination name prefix or IAM permissions"
  long_term: "Implement comprehensive logging with filtering and analysis"
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
  - "NEVER suggest disabling WAF to fix access issues"
  - "NEVER suggest removing all WAF rules"
  - "NEVER suggest allowing all IPs to bypass rate limiting"

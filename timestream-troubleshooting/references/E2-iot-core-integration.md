---
title: "E2 — IoT Core Integration Issues"
description: "Diagnose Timestream integration with AWS IoT Core"
status: active
severity: MEDIUM
triggers:
  - "IoT Core Timestream"
  - "IoT rule action"
  - "IoT data not in Timestream"
  - "IoT Timestream action"
owner: devops-agent
objective: "Identify and resolve IoT Core to Timestream integration issues"
context: "IoT Core can write data to Timestream via IoT rule actions. The rule action requires an IAM role with Timestream write permissions. The rule SQL selects data from MQTT topics and maps fields to Timestream dimensions and measures."
---

## Phase 1 — Triage

MUST:
- Check the IoT rule configuration: `aws iot get-topic-rule --rule-name <rule-name>`
- Verify the Timestream action is configured in the rule
- Check the IAM role used by the rule action: verify timestream:WriteRecords and timestream:DescribeEndpoints permissions
- Check IoT rule error actions and CloudWatch logs for failures

SHOULD:
- Verify the target Timestream database and table exist
- Check the dimension and measure mapping in the rule action
- Test the IoT rule SQL with sample MQTT messages

MAY:
- Check CloudWatch IoT rule metrics for error counts
- Verify MQTT messages are being published to the expected topic

## Phase 2 — Remediate

MUST:
- Fix the IAM role permissions for the Timestream rule action
- Ensure the target database and table exist
- Fix dimension and measure mappings in the rule action

SHOULD:
- Configure an error action to capture failed writes
- Test with a simple MQTT message before production deployment

MAY:
- Set up CloudWatch alarms for IoT rule failures
- Monitor Timestream write metrics from IoT

## Common Issues

- symptoms: "IoT rule action fails silently"
  diagnosis: "No error action configured — failures are not captured."
  resolution: "Add an error action (e.g., CloudWatch Logs or S3) to the IoT rule."

- symptoms: "AccessDeniedException in IoT rule logs"
  diagnosis: "IAM role missing Timestream write permissions."
  resolution: "Add timestream:WriteRecords and timestream:DescribeEndpoints to the role."

- symptoms: "Data written but dimensions/measures wrong"
  diagnosis: "Incorrect field mapping in the Timestream rule action."
  resolution: "Review and fix the dimension and measure mappings in the rule action."

## Output Format

```yaml
root_cause: "iot_core_integration_issue — <specific_cause>"
evidence:
  - type: rule_config
    content: "<IoT rule and Timestream action configuration>"
  - type: iam_role
    content: "<IAM role permissions>"
  - type: error_logs
    content: "<IoT rule error logs>"
severity: MEDIUM
mitigation:
  immediate: "Fix rule action configuration and permissions"
  long_term: "Monitor IoT-Timestream pipeline with error actions and alarms"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Time-series data may contain IoT sensor readings and telemetry
- Query results may reveal operational patterns and metrics
- Scheduled query outputs may aggregate sensitive measurements
- Classification: CONFIDENTIAL - operational telemetry and measurement data

## Prohibited Actions

- NEVER delete Timestream tables containing production time-series data
- NEVER reduce memory store retention below the current data window
- NEVER modify magnetic store retention without data lifecycle review
- NEVER cancel scheduled queries that feed production dashboards

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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

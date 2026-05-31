---
title: "F1 — Access Log Issues"
description: "Diagnose Verified Access access logging configuration and analysis problems"
status: active
severity: MEDIUM
triggers:
  - "access logs missing"
  - "logging not working"
  - "cannot see logs"
  - "log configuration"
owner: devops-agent
objective: "Identify and resolve access logging issues"
context: "Verified Access access logs capture policy evaluation results, user identity, device context, and request details. Logging is NOT enabled by default. Logs can go to CloudWatch Logs, S3, or Kinesis Data Firehose. Logging is essential for troubleshooting and compliance."
---

## Phase 1 — Triage

MUST:
- Check logging configuration: `aws ec2 describe-verified-access-instance-logging-configurations --verified-access-instance-ids <instance-id>`
- Verify logging is enabled
- Check log destination (CloudWatch Logs group, S3 bucket, or Firehose)
- Verify IAM permissions for log delivery

SHOULD:
- Check if log group/bucket exists and is accessible
- Verify log entries are appearing: `aws logs filter-log-events --log-group-name <group> --limit 10`
- Check for log delivery errors

MAY:
- Review log format and available fields
- Check log retention settings

## Phase 2 — Remediate

MUST:
- Enable logging: `aws ec2 modify-verified-access-instance-logging-configuration --verified-access-instance-id <id> --access-logs CloudWatchLogs={Enabled=true,LogGroup=<group>}`
- Ensure log destination exists and has correct permissions
- Verify logs appear after enabling

SHOULD:
- Configure appropriate log retention
- Set up log-based alerts for security events

MAY:
- Enable multiple log destinations for redundancy
- Implement log analysis dashboards

## Common Issues

- symptoms: "No logs appearing"
  diagnosis: "Logging not enabled on the Verified Access instance."
  resolution: "Enable logging with modify-verified-access-instance-logging-configuration."

- symptoms: "Logging enabled but no entries"
  diagnosis: "No traffic through Verified Access or log delivery permissions."
  resolution: "Generate test traffic. Check IAM permissions for log delivery."

- symptoms: "Logs missing context data"
  diagnosis: "Trust provider not supplying context or logging configuration incomplete."
  resolution: "Verify trust provider is attached and configured correctly."

## Output Format

```yaml
root_cause: "access_logs — <specific_cause>"
evidence:
  - type: logging_config
    content: "<logging configuration>"
  - type: log_destination
    content: "<log group/bucket status>"
  - type: sample_logs
    content: "<recent log entries>"
severity: MEDIUM
mitigation:
  immediate: "Enable and verify logging"
  long_term: "Implement log analysis and alerting"
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

- Trust provider configurations contain identity provider details
- Access logs reveal user access patterns and device information
- Cedar policies define authorization logic for applications
- Classification: CONFIDENTIAL - access control policies and user activity data

## Prohibited Actions

- NEVER modify trust provider configurations during active user sessions
- NEVER change Verified Access policies that could lock out production users
- NEVER delete endpoints serving active application traffic
- NEVER modify Cedar policies without testing in a non-production environment first

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

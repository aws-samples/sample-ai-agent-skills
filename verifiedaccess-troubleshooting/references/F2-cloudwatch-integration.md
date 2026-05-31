---
title: "F2 — CloudWatch Integration Issues"
description: "Diagnose CloudWatch Logs integration problems for Verified Access"
status: active
severity: MEDIUM
triggers:
  - "CloudWatch logs"
  - "log group error"
  - "log delivery failed"
  - "CloudWatch integration"
owner: devops-agent
objective: "Identify and resolve CloudWatch Logs integration issues"
context: "CloudWatch Logs is the most common destination for Verified Access access logs. Integration requires a log group and proper IAM permissions for log delivery. Log group must exist before enabling logging. Retention policies control log storage costs."
---

## Phase 1 — Triage

MUST:
- Verify log group exists: `aws logs describe-log-groups --log-group-name-prefix <group-name>`
- Check logging configuration references correct log group
- Verify IAM permissions for Verified Access to write to CloudWatch Logs
- Check for recent log entries: `aws logs filter-log-events --log-group-name <group> --start-time <epoch-ms> --limit 10`

SHOULD:
- Check log group retention policy
- Verify no resource policy blocking log delivery
- Check CloudWatch Logs service quotas

MAY:
- Set up metric filters for access patterns
- Create CloudWatch dashboards for access monitoring

## Phase 2 — Remediate

MUST:
- Create log group if missing: `aws logs create-log-group --log-group-name /aws/verified-access/<instance-id>`
- Ensure Verified Access has permission to write logs (service-linked role)
- Update logging config to use correct log group

SHOULD:
- Set retention policy: `aws logs put-retention-policy --log-group-name <group> --retention-in-days 90`
- Create metric filters for DENY events

MAY:
- Set up CloudWatch alarms for unusual access patterns
- Export logs to S3 for long-term retention

## Common Issues

- symptoms: "Log group not receiving entries"
  diagnosis: "Log group name mismatch or permissions issue."
  resolution: "Verify log group name in logging config. Check IAM permissions."

- symptoms: "Log delivery throttled"
  diagnosis: "CloudWatch Logs ingestion rate exceeded."
  resolution: "Check CloudWatch Logs quotas. Request increase if needed."

- symptoms: "Old logs missing"
  diagnosis: "Retention policy deleting old logs."
  resolution: "Increase retention period or export to S3 before expiration."

## Output Format

```yaml
root_cause: "cloudwatch_integration — <specific_cause>"
evidence:
  - type: log_group
    content: "<log group status>"
  - type: logging_config
    content: "<Verified Access logging configuration>"
  - type: permissions
    content: "<IAM permissions for log delivery>"
severity: MEDIUM
mitigation:
  immediate: "Fix log group and permissions"
  long_term: "Implement log retention and monitoring"
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

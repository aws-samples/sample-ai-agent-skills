---
title: "D2 — Log Correlation Errors"
description: "Diagnose CloudWatch Application Signals log correlation failures"
status: active
severity: MEDIUM
triggers: ["logs not correlated", "log group missing", "trace ID not in logs", "log correlation failed"]
owner: devops-agent
objective: "Resolve log correlation issues between Application Signals, X-Ray traces, and CloudWatch Logs"
context: "Log correlation requires trace IDs to be injected into application logs and logs to be sent to CloudWatch Logs."
---

## Phase 1 — Triage

MUST:
- Check log groups: `aws logs describe-log-groups --log-group-name-prefix '/aws/application-signals'`
- Verify application logs contain trace IDs (search for X-Ray trace ID format)
- Check CloudWatch Logs agent configuration

SHOULD:
- Search logs for a known trace ID: `aws logs filter-log-events --log-group-name <group> --filter-pattern '<trace-id>'`
- Verify ADOT agent injects trace IDs into logging context

MAY:
- Check application logging framework configuration (Log4j, logback, etc.)
- Review log format for trace ID field presence

## Phase 2 — Remediate

MUST:
- Configure application logging to include trace IDs in log output
- Ensure logs are sent to CloudWatch Logs
- Verify log group names follow the expected pattern for correlation

SHOULD:
- Use structured logging (JSON) with trace_id and span_id fields
- Configure ADOT agent to inject trace context into logging MDC/context

MAY:
- Set up CloudWatch Logs Insights queries for trace-based log analysis
- Configure log retention policies for cost management

## Common Issues

- symptoms: "Cannot navigate from trace to logs in Application Signals"
  diagnosis: "Application logs do not contain trace IDs or logs are not in CloudWatch Logs."
  resolution: "Configure logging framework to include trace IDs. Ensure CloudWatch agent sends logs to CloudWatch Logs."

- symptoms: "Trace IDs in logs but correlation still fails"
  diagnosis: "Log group name does not match the expected pattern for Application Signals correlation."
  resolution: "Verify log group naming convention. Application Signals expects specific log group patterns for auto-correlation."

## Safety Ratings

```
safety_ratings:
  - "logs describe-log-groups: GREEN — read-only log group listing"
  - "logs filter-log-events: GREEN — read-only log search"
  - "Logging configuration changes: YELLOW — affects log output format"
  - "Log group creation: YELLOW — creates CloudWatch Logs resources with cost"
```

## Escalation Conditions

- Log correlation fails despite trace IDs being present in logs
- Application logging framework does not support trace context injection
- Log volume is too high for cost-effective CloudWatch Logs storage
- Cross-account log correlation required

## Data Sensitivity

- `aws logs describe-log-groups`: LOW — returns log group metadata
- `aws logs filter-log-events`: HIGH — may contain application data and PII
- Application logs: HIGH — may contain sensitive request/response data
- Trace IDs: LOW — contain correlation identifiers only

## Prohibited Actions

- NEVER search production logs without understanding data sensitivity
- NEVER modify log retention policies without cost and compliance review
- NEVER disable application logging to troubleshoot correlation
- NEVER expose trace IDs in user-facing responses

## Phase 3 — Rollback

1. Revert logging configuration changes if they caused issues
2. Restore original log format if structured logging changes broke log parsing
3. Verify application logs continue to flow to CloudWatch Logs
4. Check that existing log-based alerts and dashboards still function
5. Document the log correlation configuration attempted

## Output Format

```yaml
root_cause: "log_correlation — <specific_cause>"
evidence:
  - type: log_error
    content: "<correlation details>"
severity: MEDIUM
mitigation:
  immediate: "Fix logging configuration"
  long_term: "Standardize structured logging with trace context"
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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"

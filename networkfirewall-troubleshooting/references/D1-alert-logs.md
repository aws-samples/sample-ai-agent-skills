---
title: "D1 — Alert Log Issues"
description: "Diagnose alert logging configuration and analysis problems"
status: active
severity: MEDIUM
triggers:
  - "alert logs missing"
  - "no alerts"
  - "alert logging"
  - "rule alerts"
owner: devops-agent
objective: "Identify and resolve alert logging issues"
context: "Alert logs capture entries when stateful rules match with alert or drop actions. Logging must be explicitly enabled. Logs can go to CloudWatch Logs, S3, or Kinesis Data Firehose. Alert logs are essential for security monitoring and rule tuning."
---

## Phase 1 — Triage

MUST:
- Check logging configuration: `aws network-firewall describe-logging-configuration --firewall-arn <firewall-arn>`
- Verify ALERT log type is enabled
- Check log destination exists and is accessible
- Verify stateful rules have alert or drop actions

SHOULD:
- Check for recent alert entries: `aws logs filter-log-events --log-group-name <group> --filter-pattern "{ $.event.alert }" --limit 10`
- Verify traffic is reaching the stateful engine (check stateless forwarding)
- Check IAM permissions for log delivery

MAY:
- Check log delivery latency
- Review alert log format and fields

## Phase 2 — Remediate

MUST:
- Enable alert logging: `aws network-firewall update-logging-configuration --firewall-arn <arn> --logging-configuration LogDestinationConfigs=[{LogType=ALERT,LogDestinationType=CloudWatchLogs,LogDestination={logGroup=<group>}}]`
- Ensure log destination exists
- Verify stateful rules generate alerts

SHOULD:
- Set up CloudWatch metric filters for alert patterns
- Configure log retention policies

MAY:
- Create CloudWatch dashboards for alert monitoring
- Implement automated alert response

## Common Issues

- symptoms: "No alert logs appearing"
  diagnosis: "Alert logging not enabled or no stateful rules matching."
  resolution: "Enable alert logging. Verify stateful rules have alert/drop actions."

- symptoms: "Alert logs enabled but empty"
  diagnosis: "Traffic not reaching stateful engine or rules not matching."
  resolution: "Check stateless forwarding. Verify rule syntax matches traffic."

- symptoms: "Log delivery delayed"
  diagnosis: "Log destination throttling or permissions issue."
  resolution: "Check CloudWatch Logs quotas. Verify IAM permissions."

## Output Format

```yaml
root_cause: "alert_logs — <specific_cause>"
evidence:
  - type: logging_config
    content: "<alert logging configuration>"
  - type: log_entries
    content: "<recent alert log entries>"
  - type: stateful_rules
    content: "<rules with alert/drop actions>"
severity: MEDIUM
mitigation:
  immediate: "Enable and verify alert logging"
  long_term: "Implement alert monitoring and automated response"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete or modify firewall rules that are actively filtering production traffic without a tested rollback plan
- **NEVER** change route table associations for firewall endpoints without verifying traffic flow continuity
- **NEVER** disable TLS inspection on endpoints monitoring compliance-regulated traffic
- **NEVER** remove stateful rule groups from a firewall policy during active threat mitigation
- **NEVER** modify asymmetric routing configurations without confirming bidirectional traffic inspection

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
  - "NEVER suggest disabling stateful inspection"
  - "NEVER suggest removing all firewall rules"
  - "NEVER suggest bypassing TLS inspection without review"

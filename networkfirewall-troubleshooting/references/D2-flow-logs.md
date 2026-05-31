---
title: "D2 — Flow Log Issues"
description: "Diagnose flow logging configuration and analysis problems"
status: active
severity: MEDIUM
triggers:
  - "flow logs missing"
  - "flow logging"
  - "traffic visibility"
  - "flow records"
owner: devops-agent
objective: "Identify and resolve flow logging issues"
context: "Flow logs capture all traffic flow records through the firewall, regardless of rule action. They provide visibility into traffic patterns, volumes, and firewall decisions. Flow logs must be explicitly enabled and are separate from alert logs."
---

## Phase 1 — Triage

MUST:
- Check logging configuration: `aws network-firewall describe-logging-configuration --firewall-arn <firewall-arn>`
- Verify FLOW log type is enabled
- Check log destination exists and is accessible
- Verify traffic is flowing through the firewall

SHOULD:
- Check for recent flow entries
- Verify log destination permissions
- Check log format and available fields

MAY:
- Compare flow logs with VPC Flow Logs for completeness
- Check log delivery latency

## Phase 2 — Remediate

MUST:
- Enable flow logging: `aws network-firewall update-logging-configuration --firewall-arn <arn> --logging-configuration LogDestinationConfigs=[{LogType=FLOW,LogDestinationType=S3,LogDestination={bucketName=<bucket>,prefix=<prefix>}}]`
- Ensure log destination exists with correct permissions

SHOULD:
- Use S3 for flow logs (high volume) and CloudWatch for alert logs
- Configure lifecycle policies for S3 flow logs

MAY:
- Use Athena to query S3 flow logs
- Create dashboards for traffic analysis

## Common Issues

- symptoms: "No flow logs appearing"
  diagnosis: "Flow logging not enabled or no traffic through firewall."
  resolution: "Enable flow logging. Verify routing sends traffic through firewall."

- symptoms: "Flow logs incomplete"
  diagnosis: "Some traffic bypassing firewall due to routing."
  resolution: "Check route tables. Ensure all traffic paths go through firewall."

- symptoms: "Flow log volume very high"
  diagnosis: "All traffic generates flow records."
  resolution: "Use S3 for storage. Implement lifecycle policies. Consider sampling."

## Output Format

```yaml
root_cause: "flow_logs — <specific_cause>"
evidence:
  - type: logging_config
    content: "<flow logging configuration>"
  - type: log_entries
    content: "<sample flow log entries>"
severity: MEDIUM
mitigation:
  immediate: "Enable and verify flow logging"
  long_term: "Implement log analysis and retention policies"
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

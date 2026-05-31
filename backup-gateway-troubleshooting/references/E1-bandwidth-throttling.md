---
title: "E1 — Bandwidth Throttling"
description: "Diagnose bandwidth throttling and throughput issues"
status: active
severity: MEDIUM
triggers: ["bandwidth throttling", "slow backup", "throughput low", "upload speed slow"]
owner: devops-agent
objective: "Resolve bandwidth throttling and throughput issues for Backup Gateway"
context: "Gateway bandwidth is shared across all operations. Throughput depends on gateway resources, network capacity, and concurrent operations."
---

## Phase 1 — Triage

MUST:
- Check gateway throughput metrics: `aws cloudwatch get-metric-statistics --namespace AWS/BackupGateway --metric-name GatewayBytesUploaded --dimensions Name=GatewayArn,Value=<arn> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Count concurrent backup/restore operations: `aws backup list-backup-jobs --by-state RUNNING` and `aws backup list-restore-jobs --by-status RUNNING`
- Verify gateway VM resource utilization

SHOULD:
- Compare throughput against baseline or expected values
- Check network bandwidth between gateway and AWS

MAY:
- Monitor network utilization on the gateway VM's network interface
- Check for QoS policies affecting gateway traffic

## Phase 2 — Remediate

MUST:
- Reduce concurrent operations if bandwidth is saturated
- Ensure gateway VM has adequate network bandwidth allocation
- Verify no network throttling policies affect gateway traffic

SHOULD:
- Stagger backup schedules to reduce peak bandwidth demand
- Increase gateway VM network allocation if possible

MAY:
- Implement bandwidth scheduling for backup windows
- Consider AWS Direct Connect for higher throughput
- Deploy additional gateways to distribute load

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "backup list-backup-jobs: GREEN — read-only job listing"
  - "backup list-restore-jobs: GREEN — read-only job listing"
  - "gateway VM network reconfiguration: YELLOW — may interrupt active operations"
```

## Escalation Conditions

- Bandwidth is insufficient to complete backups within the backup window
- Network infrastructure changes require network team involvement
- Gateway VM cannot be allocated more network bandwidth
- Throughput degradation affects backup SLA compliance

## Data Sensitivity

- CloudWatch metrics: LOW — reveals throughput statistics
- `aws backup list-backup-jobs`: MEDIUM — reveals backup job details
- Network utilization data: LOW — reveals bandwidth consumption patterns

## Prohibited Actions

- NEVER remove QoS policies without understanding their purpose
- NEVER increase gateway bandwidth at the expense of production traffic
- NEVER cancel running backup jobs to free bandwidth without approval
- NEVER assume bandwidth issues are always network-related — check gateway resources

## Phase 3 — Rollback

1. If backup schedules were modified, restore original schedules
2. If gateway VM network was reconfigured, revert if throughput did not improve
3. If QoS policies were modified, restore original settings
4. Verify all backup jobs complete within the backup window
5. Document the bandwidth baseline and optimization changes

## Common Issues

- symptoms: "Backup jobs take 3x longer than expected"
  diagnosis: "Multiple concurrent backups saturating gateway bandwidth."
  resolution: "Stagger backup schedules. Reduce concurrent job count. Verify gateway VM has adequate network bandwidth."

## Output Format

```yaml
root_cause: "bandwidth_throttling — <specific_cause>"
evidence:
  - type: throughput_data
    content: "<bandwidth metrics>"
severity: MEDIUM
mitigation:
  immediate: "Reduce contention"
  long_term: "Right-size bandwidth and schedule backups"
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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"

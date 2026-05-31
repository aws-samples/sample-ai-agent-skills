---
title: "D2 — Restore Performance"
description: "Diagnose slow restore performance"
status: active
severity: MEDIUM
triggers: ["restore slow", "restore taking too long", "poor restore performance", "restore throughput low"]
owner: devops-agent
objective: "Resolve restore performance issues in Backup Gateway"
context: "Restore performance depends on gateway resources, network bandwidth, and target storage I/O. Bandwidth is shared across operations."
---

## Phase 1 — Triage

MUST:
- Check restore job progress: `aws backup describe-restore-job --restore-job-id <job-id>`
- Monitor gateway throughput: `aws cloudwatch get-metric-statistics --namespace AWS/BackupGateway --metric-name GatewayBytesDownloaded --dimensions Name=GatewayArn,Value=<arn> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check for concurrent operations on the gateway

SHOULD:
- Verify gateway VM resource utilization (CPU, memory, disk I/O)
- Check network bandwidth between gateway and AWS

MAY:
- Compare current throughput against baseline performance
- Check target datastore I/O performance

## Phase 2 — Remediate

MUST:
- Reduce concurrent operations if bandwidth is saturated
- Ensure gateway VM has adequate resources
- Verify network path has sufficient bandwidth

SHOULD:
- Schedule restores during off-peak hours
- Increase gateway VM resources if consistently under-provisioned

MAY:
- Use AWS Direct Connect for higher bandwidth to AWS
- Optimize target datastore performance

## Safety Ratings

```
safety_ratings:
  - "backup describe-restore-job: GREEN — read-only job details"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "gateway VM resource adjustment: YELLOW — requires VM reconfiguration"
```

## Escalation Conditions

- Restore performance does not meet RTO requirements
- Network bandwidth is insufficient and requires infrastructure changes
- Gateway VM cannot be resized due to ESXi host constraints
- Restore performance degrades over time without explanation

## Data Sensitivity

- `aws backup describe-restore-job`: MEDIUM — reveals restore details
- CloudWatch metrics: LOW — reveals throughput statistics
- Network bandwidth data: LOW — reveals capacity information

## Prohibited Actions

- NEVER cancel a running restore to start a new one without understanding the impact
- NEVER reduce gateway VM resources to free up ESXi host capacity
- NEVER assume restore performance equals backup performance
- NEVER skip network bandwidth verification when diagnosing slow restores

## Phase 3 — Rollback

1. If gateway VM resources were changed, revert if performance did not improve
2. If concurrent operations were paused, resume them after the restore completes
3. Verify backup schedules are not impacted by restore operations
4. Document the performance baseline for future comparison
5. Restore original gateway VM configuration if changes were ineffective

## Common Issues

- symptoms: "Restore is progressing but estimated completion time is hours away"
  diagnosis: "Bandwidth contention from concurrent backups or insufficient gateway resources."
  resolution: "Pause non-critical backup jobs. Verify gateway VM has adequate CPU and memory. Check network bandwidth."

## Output Format

```yaml
root_cause: "restore_performance — <specific_cause>"
evidence:
  - type: performance_data
    content: "<throughput metrics>"
severity: MEDIUM
mitigation:
  immediate: "Reduce contention and optimize resources"
  long_term: "Right-size gateway and network capacity"
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

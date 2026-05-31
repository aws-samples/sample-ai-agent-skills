---
title: "E3 — EBS IOPS / Throughput Throttling"
description: "Diagnose EBS performance throttling due to IOPS or throughput limits"
status: active
severity: MEDIUM
triggers:
  - "VolumeQueueLength.*high"
  - "BurstBalance.*0"
  - "VolumeThroughputPercentage.*100"
owner: devops-agent
objective: "Identify the throttling source and restore volume performance"
context: "EBS throttling occurs at two levels: volume-level (IOPS/throughput per volume type) and instance-level (EBS-optimized bandwidth). gp2 has burst credits. gp3/io1/io2 have provisioned limits. Instance EBS bandwidth is shared across all attached volumes."
---

## Phase 1 — Triage

MUST:
- Check volume type and provisioned performance: `aws ec2 describe-volumes --volume-ids <vol-id>`
- Check CloudWatch: VolumeReadOps, VolumeWriteOps, VolumeQueueLength
- For gp2: check BurstBalance (0% = throttled to baseline)
- Check instance EBS bandwidth limit vs actual throughput

SHOULD:
- Compare actual IOPS against volume limits:
  - gp3: 3000 baseline, up to 16000 provisioned
  - gp2: 3 IOPS/GB (min 100, max 16000), burst to 3000
  - io1: up to 64000 IOPS (50 IOPS/GB ratio)
  - io2: up to 64000 IOPS (500 IOPS/GB ratio)
- Check if instance EBS bandwidth is the bottleneck (not the volume)

## Common Issues

- symptoms: "gp2 BurstBalance at 0%, high queue length"
  diagnosis: "gp2 burst credits exhausted. Baseline is 3 IOPS/GB."
  resolution: "Migrate to gp3 (3000 baseline IOPS, independent of size). Or increase gp2 size (1TB = 3000 baseline)."

- symptoms: "io1/io2 at provisioned IOPS limit"
  diagnosis: "Provisioned IOPS fully consumed."
  resolution: "Increase provisioned IOPS (up to 64000). Check 50:1 or 500:1 IOPS-to-GB ratio."

- symptoms: "Multiple volumes throttled simultaneously"
  diagnosis: "Instance EBS-optimized bandwidth is the bottleneck, not individual volumes."
  resolution: "Upgrade instance type for higher EBS bandwidth. Distribute I/O across fewer volumes."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-volumes, CloudWatch metrics: GREEN — read-only"
  - "describe-instance-types for EBS bandwidth: GREEN — read-only"
  - "Migrate gp2 to gp3: YELLOW — online migration, brief performance impact"
  - "Increase provisioned IOPS on io1/io2: YELLOW — cost increase, recoverable by reducing"
  - "Modify volume size to increase gp2 baseline: YELLOW — cannot shrink back, 6-hour cooldown"
  - "Upgrade instance type for EBS bandwidth: YELLOW — requires stop+start"
```

## Escalation Conditions
- IOPS throttling is causing database transaction timeouts or data corruption
- Volume is in 6-hour cooldown from recent modification
- Instance EBS bandwidth is the bottleneck and requires instance type change in production
- Multiple volumes are simultaneously throttled affecting application availability
- gp2 to gp3 migration is needed for a root volume on a production instance

## Data Sensitivity
- MEDIUM: CloudWatch EBS metrics (reveals I/O patterns, workload characteristics)
- MEDIUM: describe-volumes (reveals volume configuration, provisioned performance)
- LOW: describe-instance-types (public EBS bandwidth data)

## Prohibited Actions
- NEVER suggest reducing volume size to change type (EBS cannot be shrunk)
- NEVER suggest modifying a volume during the 6-hour cooldown period
- NEVER suggest io2 Block Express without confirming Nitro instance compatibility
- NEVER suggest removing EBS optimization from an instance

## Phase 3 — Rollback
- If volume type was changed (gp2→gp3): cannot revert directly, create new gp2 from snapshot if needed
- If provisioned IOPS were increased: reduce IOPS with modify-volume (subject to 6-hour cooldown)
- If volume size was increased: cannot shrink — create smaller volume from snapshot if needed
- If instance type was upgraded: stop, change back to original type, restart

## Output Format

```yaml
root_cause: "<gp2_burst|volume_iops_limit|volume_throughput_limit|instance_ebs_bandwidth>"
evidence:
  - type: cloudwatch_metric
    content: "<IOPS and queue length metrics>"
severity: MEDIUM
mitigation:
  immediate: "Upgrade volume type or increase provisioned IOPS"
  long_term: "Right-size volumes for sustained workload, use gp3 over gp2"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"

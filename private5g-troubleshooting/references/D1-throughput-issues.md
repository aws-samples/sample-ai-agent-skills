---
title: "D1 — Throughput Issues"
description: "Diagnose throughput degradation on Private 5G"
status: active
severity: HIGH
triggers: ["slow speed", "low throughput", "bandwidth issues", "poor performance"]
owner: devops-agent
objective: "Resolve throughput degradation on the Private 5G network"
context: "Throughput depends on CBRS spectrum allocation, radio unit placement, device count, physical environment, and RF interference."
---

## Phase 1 — Triage

MUST:
- Check network resource status: `aws private-networks list-network-resources --network-arn <arn>`
- Review throughput metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Private5G --metric-name Throughput --dimensions Name=NetworkArn,Value=<arn> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Count connected devices: `aws cloudwatch get-metric-statistics --namespace AWS/Private5G --metric-name ConnectedDevices --dimensions Name=NetworkArn,Value=<arn> --start-time <start> --end-time <end> --period 300 --statistics Maximum`

SHOULD:
- Compare current throughput against baseline measurements
- Check for new sources of RF interference
- Verify radio unit health and signal strength

MAY:
- Run speed tests from multiple locations within coverage area
- Check VPC network throughput for bottlenecks

## Phase 2 — Remediate

MUST:
- Identify if throughput issue is RF-related or network-related
- Reduce device count per radio unit if overloaded
- Verify VPC security groups and NACLs are not throttling traffic

SHOULD:
- Reposition radio units for better coverage if physical obstacles are identified
- Add additional radio units for high-density areas
- Check VPC subnet ENI throughput limits

MAY:
- Schedule high-bandwidth operations during off-peak hours
- Implement QoS policies at the application level

## Safety Ratings

```
safety_ratings:
  - "list-network-resources: GREEN — read-only enumeration"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "radio unit repositioning: YELLOW — causes temporary service interruption"
  - "security group modification: YELLOW — affects network access rules"
```

## Escalation Conditions

- Throughput consistently below 10% of expected baseline
- All radio units showing degraded performance simultaneously
- Throughput issues correlate with specific times suggesting external interference
- VPC-side throughput is normal but device-side is degraded

## Data Sensitivity

- `aws cloudwatch get-metric-statistics`: LOW — aggregate performance metrics
- Throughput test results: LOW — network performance data
- Device count metrics: MEDIUM — reveals network utilization patterns

## Prohibited Actions

- NEVER quote theoretical 5G speeds as expected Private 5G performance
- NEVER modify radio unit hardware or antennas
- NEVER disable security groups to troubleshoot throughput
- NEVER promise specific throughput numbers — CBRS performance varies by environment

## Phase 3 — Rollback

1. If security groups were modified, restore original rules
2. If radio units were repositioned, document new and old positions
3. Verify throughput returns to baseline after changes
4. Confirm all devices maintain connectivity after any network changes
5. Document throughput measurements before and after remediation

## Common Issues

- symptoms: "Throughput drops during business hours"
  diagnosis: "Too many devices sharing the same radio unit bandwidth."
  resolution: "Add additional radio units to distribute load. Consider device scheduling."

- symptoms: "Throughput is low even with a single device"
  diagnosis: "Physical obstacles or RF interference between device and radio unit."
  resolution: "Move device closer to radio unit. Check for new interference sources."

## Output Format

```yaml
root_cause: "throughput — <specific_cause>"
evidence:
  - type: performance_metrics
    content: "<throughput data>"
severity: HIGH
mitigation:
  immediate: "Address throughput bottleneck"
  long_term: "Optimize radio unit placement and capacity planning"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"

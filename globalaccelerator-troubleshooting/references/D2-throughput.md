---
title: "D2 — Global Accelerator Throughput Issues"
description: "Diagnose throughput degradation through Global Accelerator"
status: active
severity: HIGH
triggers:
  - "low throughput"
  - "bandwidth issues"
  - "throughput degradation"
  - "slow transfer"
owner: devops-agent
objective: "Identify and resolve throughput issues in Global Accelerator"
context: "Global Accelerator uses the AWS global network for optimized throughput. Throughput issues can stem from endpoint capacity, endpoint group configuration, or network path problems."
---

## Phase 1 — Triage

MUST:
- Check processed bytes metrics: `aws cloudwatch get-metric-statistics --namespace AWS/GlobalAccelerator --metric-name ProcessedBytesIn --dimensions Name=Accelerator,Value=<accelerator-arn> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check outbound bytes: `aws cloudwatch get-metric-statistics --namespace AWS/GlobalAccelerator --metric-name ProcessedBytesOut --dimensions Name=Accelerator,Value=<accelerator-arn> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify endpoint health and capacity: `aws globalaccelerator describe-endpoint-group --endpoint-group-arn <endpoint-group-arn>`
- Check new flow count for connection patterns: `aws cloudwatch get-metric-statistics --namespace AWS/GlobalAccelerator --metric-name NewFlowCount --dimensions Name=Accelerator,Value=<accelerator-arn> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check if endpoint (ALB/NLB) is hitting its own throughput limits
- Verify endpoint instance types have sufficient network bandwidth
- Check for TCP window size or MTU issues

MAY:
- Enable flow logs to analyze traffic patterns
- Compare throughput during peak vs off-peak hours
- Check if DDoS attack is consuming bandwidth

## Phase 2 — Remediate

MUST:
- Scale endpoint capacity if endpoints are the bottleneck
- Ensure endpoint security groups allow sufficient traffic
- Verify endpoint group has enough healthy endpoints to handle load

SHOULD:
- Distribute traffic across multiple endpoint groups/regions
- Use appropriate endpoint types (NLB for high-throughput TCP)

MAY:
- Consider using multiple accelerators for very high throughput requirements
- Enable Shield Advanced for DDoS-related throughput issues

## Common Issues

- symptoms: "Throughput drops during peak hours"
  diagnosis: "Endpoint capacity is insufficient for peak traffic."
  resolution: "Scale endpoint instances or add more endpoints to the group."

- symptoms: "Throughput lower than expected for large transfers"
  diagnosis: "TCP window size or MTU issues between GA and endpoint."
  resolution: "Verify MTU settings and TCP optimization on endpoints."

- symptoms: "Sudden throughput drop"
  diagnosis: "Endpoint became unhealthy, traffic shifted to fewer endpoints."
  resolution: "Fix unhealthy endpoints to restore full capacity."

## Output Format

```yaml
root_cause: "throughput — <specific_cause>"
evidence:
  - type: processed_bytes
    content: "<inbound/outbound byte metrics>"
  - type: flow_count
    content: "<new flow count metrics>"
  - type: endpoint_capacity
    content: "<endpoint health and capacity>"
severity: HIGH
mitigation:
  immediate: "Scale endpoints or fix unhealthy endpoints"
  long_term: "Implement auto-scaling and throughput monitoring"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never disable or delete an accelerator serving production traffic without confirming failover
- Never remove endpoint groups without verifying traffic has been drained
- Never modify static IP associations on active accelerators
- Never change routing configurations during active incidents without approval
- Never delete flow logs that may be needed for audit or investigation

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Re-enable previous endpoint group configurations if traffic routing degrades
- Restore traffic dial settings to previous values if performance worsens
- Revert client affinity settings if session persistence breaks

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

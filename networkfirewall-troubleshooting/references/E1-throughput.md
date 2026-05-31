---
title: "E1 — Throughput Issues"
description: "Diagnose throughput degradation through Network Firewall"
status: active
severity: MEDIUM
triggers:
  - "throughput degradation"
  - "bandwidth limited"
  - "firewall bottleneck"
  - "slow throughput"
owner: devops-agent
objective: "Identify and resolve throughput issues through Network Firewall"
context: "Network Firewall throughput depends on rule complexity, TLS inspection, traffic patterns, and endpoint capacity. Each firewall endpoint supports up to 100 Gbps. Complex Suricata rules and TLS inspection reduce throughput. Throughput scales with the number of endpoints (AZs)."
---

## Phase 1 — Triage

MUST:
- Check firewall metrics: `aws cloudwatch get-metric-statistics --namespace AWS/NetworkFirewall --metric-name Packets --dimensions Name=FirewallName,Value=<name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check DroppedPackets metric for capacity-related drops
- Verify number of firewall endpoints (AZs)
- Check if TLS inspection is enabled (reduces throughput)

SHOULD:
- Check rule group complexity (number and type of rules)
- Monitor per-AZ traffic distribution
- Compare throughput before and after firewall

MAY:
- Check for traffic spikes causing temporary throughput issues
- Review Suricata rule efficiency

## Phase 2 — Remediate

MUST:
- Add firewall endpoints in additional AZs to increase capacity
- Optimize complex Suricata rules
- Evaluate TLS inspection impact on throughput

SHOULD:
- Use stateless rules for simple filtering (faster than stateful)
- Reduce unnecessary rule groups
- Distribute traffic evenly across AZs

MAY:
- Request throughput limit increase if needed
- Implement traffic shaping before the firewall

## Common Issues

- symptoms: "Throughput drops after enabling TLS inspection"
  diagnosis: "TLS inspection adds significant processing overhead."
  resolution: "Expected behavior. Limit TLS inspection to critical traffic. Add capacity."

- symptoms: "Throughput lower than expected"
  diagnosis: "Complex Suricata rules or too many rule groups."
  resolution: "Optimize rules. Remove unnecessary rule groups. Use stateless where possible."

- symptoms: "Throughput varies by AZ"
  diagnosis: "Uneven traffic distribution across AZs."
  resolution: "Verify routing distributes traffic evenly. Add endpoints in busy AZs."

## Output Format

```yaml
root_cause: "throughput — <specific_cause>"
evidence:
  - type: firewall_metrics
    content: "<packets, bytes, drops>"
  - type: endpoint_count
    content: "<number of AZs/endpoints>"
  - type: rule_complexity
    content: "<rule group count and types>"
severity: MEDIUM
mitigation:
  immediate: "Optimize rules or add capacity"
  long_term: "Monitor throughput and plan capacity"
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

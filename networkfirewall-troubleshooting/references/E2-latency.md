---
title: "E2 — Latency Issues"
description: "Diagnose latency increase caused by Network Firewall"
status: active
severity: MEDIUM
triggers:
  - "latency increase"
  - "slow connections"
  - "firewall latency"
  - "added delay"
owner: devops-agent
objective: "Identify and resolve latency issues introduced by Network Firewall"
context: "Network Firewall adds some latency to traffic flows. Typical latency is sub-millisecond for stateless and low single-digit milliseconds for stateful inspection. TLS inspection adds more latency. Cross-AZ routing to firewall endpoints adds network latency. Excessive latency usually indicates configuration issues."
---

## Phase 1 — Triage

MUST:
- Measure latency with and without firewall (bypass test)
- Check if traffic is crossing AZs to reach firewall endpoints
- Check if TLS inspection is enabled
- Verify rule complexity is not excessive

SHOULD:
- Check CloudWatch metrics for processing time
- Verify traffic is routed to the local AZ firewall endpoint
- Check for connection timeouts vs general slowness

MAY:
- Use traceroute to identify latency sources
- Check for network congestion on the path

## Phase 2 — Remediate

MUST:
- Ensure traffic routes to LOCAL AZ firewall endpoint (avoid cross-AZ)
- Optimize complex rules that add processing time
- Evaluate TLS inspection necessity for latency-sensitive traffic

SHOULD:
- Use stateless rules for simple filtering (lower latency than stateful)
- Minimize the number of rule groups evaluated

MAY:
- Exempt latency-sensitive traffic from deep inspection
- Implement latency monitoring and alerting

## Common Issues

- symptoms: "Significant latency increase after enabling firewall"
  diagnosis: "Cross-AZ routing or TLS inspection adding latency."
  resolution: "Route to local AZ endpoint. Evaluate TLS inspection impact."

- symptoms: "Latency spikes during peak traffic"
  diagnosis: "Firewall processing capacity temporarily exceeded."
  resolution: "Add endpoints in additional AZs. Optimize rules."

- symptoms: "Connection timeouts through firewall"
  diagnosis: "Stateful rule timeout or asymmetric routing."
  resolution: "Check for asymmetric routing. Adjust stateful rule timeouts."

## Output Format

```yaml
root_cause: "latency — <specific_cause>"
evidence:
  - type: latency_measurement
    content: "<latency with and without firewall>"
  - type: routing_path
    content: "<traffic path through firewall>"
  - type: tls_inspection
    content: "<TLS inspection status>"
severity: MEDIUM
mitigation:
  immediate: "Fix cross-AZ routing or optimize rules"
  long_term: "Monitor latency and optimize firewall configuration"
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

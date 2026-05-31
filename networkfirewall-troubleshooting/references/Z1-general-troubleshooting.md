---
title: "Z1 — General Network Firewall Troubleshooting"
description: "Catch-all systematic investigation for Network Firewall issues"
status: active
severity: MEDIUM
triggers:
  - "Network Firewall issue"
  - "firewall problem"
  - "firewall error"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate Network Firewall issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for Network Firewall issues that don't clearly match a specific runbook. Use this as a starting point, then pivot to a specific runbook once the problem area is identified."
---

## Phase 1 — Triage

MUST:
- Check firewall status: `aws network-firewall describe-firewall --firewall-name <name>`
- Check firewall policy: `aws network-firewall describe-firewall-policy --firewall-policy-name <policy>`
- List rule groups: `aws network-firewall list-rule-groups`
- Check logging: `aws network-firewall describe-logging-configuration --firewall-arn <arn>`
- Check route tables: `aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id>`

SHOULD:
- Check CloudTrail for recent changes
- Review alert and flow logs if enabled
- Check CloudWatch metrics: Packets, DroppedPackets, PassedPackets

MAY:
- Check AWS Health Dashboard
- Review recent configuration changes

## Phase 2 — Remediate

MUST:
- Based on triage, pivot to specific runbook:
  - Firewall → A1 (creation), A2 (deployment), A3 (endpoints)
  - Rules → B1 (stateless), B2 (stateful), B3 (rule order)
  - Routing → C1 (route tables), C2 (asymmetric), C3 (inspection VPC)
  - Logging → D1 (alert logs), D2 (flow logs)
  - Performance → E1 (throughput), E2 (latency)
  - TLS → F1 (inspection), F2 (certificates)
  - Integration → G1 (Firewall Manager), G2 (Transit Gateway)

SHOULD:
- Enable logging if not already enabled
- Document findings

MAY:
- Set up comprehensive monitoring
- Create architecture documentation

## Common Issues

- symptoms: "Firewall not working — no specific error"
  diagnosis: "Most likely routing not configured to send traffic through firewall."
  resolution: "Check route tables first. Then firewall status → rules → logging."

- symptoms: "Firewall was working but stopped"
  diagnosis: "Recent change to routing, rules, or firewall configuration."
  resolution: "Check CloudTrail for changes. Review route tables and rule groups."

- symptoms: "Some traffic inspected but not all"
  diagnosis: "Incomplete routing or rule scope."
  resolution: "Check all subnet route tables. Verify rule groups cover all traffic types."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: firewall_status
    content: "<firewall state>"
  - type: routing
    content: "<route table configuration>"
  - type: rules
    content: "<rule group summary>"
  - type: logging
    content: "<logging configuration>"
severity: MEDIUM
general_analysis:
  firewall_state: "ready | provisioning | deleting"
  routing_configured: "yes | no | partial"
  logging_enabled: "yes | no"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive monitoring and documentation"
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

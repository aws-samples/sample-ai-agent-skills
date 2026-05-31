---
title: "B1 — Stateless Rule Issues"
description: "Diagnose stateless rule configuration and matching problems"
status: active
severity: HIGH
triggers:
  - "stateless rule not matching"
  - "stateless rule error"
  - "packet not filtered"
  - "stateless configuration"
owner: devops-agent
objective: "Identify and resolve stateless rule configuration issues"
context: "Stateless rules evaluate each packet individually without connection tracking. They match on 5-tuple (source/dest IP, source/dest port, protocol). Stateless rules are evaluated BEFORE stateful rules. Actions: pass, drop, or forward to stateful rule groups. Rules are evaluated by priority (lowest number first)."
---

## Phase 1 — Triage

MUST:
- Check stateless rule groups: `aws network-firewall describe-rule-group --rule-group-name <name> --type STATELESS`
- Verify rule matches the traffic (source/dest IP, ports, protocol)
- Check rule priority order
- Verify the firewall policy references the rule group

SHOULD:
- Check default stateless action in firewall policy
- Verify rule group is associated with the correct firewall policy
- Test with specific traffic to confirm rule matching

MAY:
- Check CloudWatch metrics: PassedPackets, DroppedPackets
- Review flow logs for traffic matching analysis

## Phase 2 — Remediate

MUST:
- Fix rule 5-tuple to match intended traffic
- Ensure rule priority is correct (lower number = higher priority)
- Verify rule group is referenced in firewall policy

SHOULD:
- Use "forward to stateful" for traffic needing deep inspection
- Set appropriate default action for unmatched traffic

MAY:
- Organize rules by function (allow known good, block known bad, forward rest)
- Document rule purpose and expected traffic

## Common Issues

- symptoms: "Stateless rule not matching traffic"
  diagnosis: "5-tuple doesn't match actual traffic or rule priority too low."
  resolution: "Verify source/dest IP, ports, protocol match. Check priority order."

- symptoms: "All traffic being dropped"
  diagnosis: "Default stateless action is drop and no pass rules match."
  resolution: "Add pass rules for legitimate traffic or change default to forward-to-stateful."

- symptoms: "Stateful rules not seeing traffic"
  diagnosis: "Stateless rules passing or dropping before forwarding to stateful."
  resolution: "Change stateless action to 'forward to stateful' for traffic needing inspection."

## Output Format

```yaml
root_cause: "stateless_rules — <specific_cause>"
evidence:
  - type: rule_config
    content: "<stateless rule definitions>"
  - type: rule_priority
    content: "<rule priority order>"
  - type: default_action
    content: "<stateless default action>"
severity: HIGH
mitigation:
  immediate: "Fix rule configuration to match intended traffic"
  long_term: "Document rules and implement change management"
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

---
title: "B3 — Rule Order Issues"
description: "Diagnose rule evaluation order and priority conflicts"
status: active
severity: HIGH
triggers:
  - "rule order"
  - "wrong rule matching"
  - "rule priority"
  - "STRICT_ORDER"
  - "DEFAULT_ACTION_ORDER"
owner: devops-agent
objective: "Identify and resolve rule evaluation order issues"
context: "Stateful rule evaluation order is set on the firewall policy: STRICT_ORDER (priority-based, lowest first) or DEFAULT_ACTION_ORDER (pass first, then drop, then alert). The mode affects which rule matches first. In STRICT_ORDER, a lower priority number rule always wins over higher. In DEFAULT_ACTION_ORDER, all pass rules are checked before any drop rules."
---

## Phase 1 — Triage

MUST:
- Check firewall policy rule order: `aws network-firewall describe-firewall-policy --firewall-policy-name <name> --query 'FirewallPolicy.StatefulEngineOptions'`
- Check rule group priorities in the policy
- Identify which rule is matching (check alert logs)
- Verify the intended rule has correct priority

SHOULD:
- List all stateful rule groups and their priorities
- Check for overlapping rules that may conflict
- Verify default stateful action

MAY:
- Map out rule evaluation flow for the traffic in question
- Check if rule order mode was recently changed

## Phase 2 — Remediate

MUST:
- Set correct rule order mode on policy
- Assign appropriate priorities to rule groups (STRICT_ORDER)
- Ensure intended rule has lower priority number than conflicting rules

SHOULD:
- In STRICT_ORDER: use priority gaps (100, 200, 300) for easy insertion
- Document rule group priority assignments and rationale

MAY:
- Reorganize rule groups for clearer evaluation flow
- Implement rule order testing procedures

## Common Issues

- symptoms: "Drop rule matching before pass rule"
  diagnosis: "In STRICT_ORDER: drop rule has lower priority number. In DEFAULT_ACTION_ORDER: this shouldn't happen."
  resolution: "STRICT_ORDER: give pass rule lower priority number. Or switch to DEFAULT_ACTION_ORDER."

- symptoms: "Pass rule allowing traffic that should be blocked"
  diagnosis: "In DEFAULT_ACTION_ORDER: pass rules always evaluate first."
  resolution: "Remove overly broad pass rule. Or switch to STRICT_ORDER for explicit ordering."

- symptoms: "Rule order mode changed and traffic behavior changed"
  diagnosis: "Switching modes changes which rules match first."
  resolution: "Review all rules under the new mode. Adjust priorities or rules accordingly."

## Output Format

```yaml
root_cause: "rule_order — <specific_cause>"
evidence:
  - type: rule_order_mode
    content: "<STRICT_ORDER or DEFAULT_ACTION_ORDER>"
  - type: rule_priorities
    content: "<rule group priorities>"
  - type: matching_rule
    content: "<which rule is actually matching>"
severity: HIGH
mitigation:
  immediate: "Fix rule priorities or evaluation mode"
  long_term: "Document rule order strategy and test changes"
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

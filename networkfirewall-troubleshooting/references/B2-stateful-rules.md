---
title: "B2 — Stateful Rule Issues"
description: "Diagnose stateful rule configuration and detection problems"
status: active
severity: HIGH
triggers:
  - "stateful rule not matching"
  - "Suricata rule error"
  - "domain filtering"
  - "stateful inspection"
owner: devops-agent
objective: "Identify and resolve stateful rule configuration issues"
context: "Stateful rules evaluate traffic flows with connection tracking. They support Suricata-compatible rules, domain lists, and 5-tuple rules. Stateful rules only see traffic forwarded by stateless rules. Rule format depends on the rule group type: Suricata string, domain list, or 5-tuple."
---

## Phase 1 — Triage

MUST:
- Check stateful rule groups: `aws network-firewall describe-rule-group --rule-group-name <name> --type STATEFUL`
- Verify rule syntax (Suricata format, domain list, or 5-tuple)
- Check if stateless rules forward traffic to stateful engine
- Verify rule group is referenced in firewall policy

SHOULD:
- Check alert logs for rule matches: `aws logs filter-log-events --log-group-name <group> --filter-pattern "alert"`
- Verify rule group type matches intended use
- Check stateful rule order mode (STRICT_ORDER vs DEFAULT_ACTION_ORDER)

MAY:
- Test Suricata rules with sample traffic
- Check for rule syntax errors in Suricata format

## Phase 2 — Remediate

MUST:
- Fix rule syntax for the rule group type
- Ensure stateless rules forward relevant traffic to stateful engine
- Verify rule group is in the firewall policy

SHOULD:
- For Suricata rules: `alert http $HOME_NET any -> $EXTERNAL_NET any (msg:"HTTP request"; sid:1; rev:1;)`
- For domain lists: configure allowed/denied domains with protocol (HTTP/HTTPS)
- Enable alert logging to verify rule matches

MAY:
- Use Suricata rule testing tools for validation
- Implement rule group versioning

## Common Issues

- symptoms: "Stateful rule not detecting traffic"
  diagnosis: "Stateless rules not forwarding traffic to stateful engine."
  resolution: "Set stateless default action to 'forward to stateful' or add forward rules."

- symptoms: "Suricata rule syntax error"
  diagnosis: "Invalid Suricata rule format."
  resolution: "Verify Suricata syntax: action protocol source port -> dest port (options; sid:N; rev:N;)"

- symptoms: "Domain filtering not blocking domains"
  diagnosis: "Domain list rule group not matching HTTPS traffic (requires SNI inspection)."
  resolution: "Ensure rule group type is domain-list with HTTPS protocol. Check SNI availability."

## Output Format

```yaml
root_cause: "stateful_rules — <specific_cause>"
evidence:
  - type: rule_config
    content: "<stateful rule definitions>"
  - type: rule_type
    content: "<Suricata, domain-list, or 5-tuple>"
  - type: alert_logs
    content: "<alert log entries>"
severity: HIGH
mitigation:
  immediate: "Fix rule configuration and verify matching"
  long_term: "Implement rule testing and alert monitoring"
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

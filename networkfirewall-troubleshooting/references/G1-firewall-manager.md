---
title: "G1 — Firewall Manager Integration Issues"
description: "Diagnose AWS Firewall Manager policy deployment problems for Network Firewall"
status: active
severity: MEDIUM
triggers:
  - "Firewall Manager"
  - "FMS policy"
  - "centralized management"
  - "policy deployment"
owner: devops-agent
objective: "Identify and resolve Firewall Manager integration issues with Network Firewall"
context: "AWS Firewall Manager can centrally manage Network Firewall policies across multiple accounts and VPCs in an AWS Organization. FMS creates and manages firewalls based on policy scope. Issues include policy deployment failures, scope misconfiguration, and rule group synchronization."
---

## Phase 1 — Triage

MUST:
- Check FMS policy: `aws fms get-policy --policy-id <policy-id>`
- Check policy compliance: `aws fms get-compliance-detail --policy-id <policy-id> --member-account <account-id>`
- Verify FMS administrator account is configured
- Check if target accounts are in scope

SHOULD:
- Verify Organization is enabled and accounts are members
- Check FMS policy scope (accounts, OUs, regions)
- Review FMS-managed firewall status in target accounts

MAY:
- Check CloudTrail for FMS policy deployment events
- Verify resource tags match policy scope

## Phase 2 — Remediate

MUST:
- Ensure FMS administrator account is set: `aws fms associate-admin-account --admin-account <account-id>`
- Fix policy scope to include target accounts/OUs
- Verify Network Firewall policy and rule groups are valid

SHOULD:
- Check compliance status for each account
- Review auto-remediation settings

MAY:
- Use FMS for consistent firewall deployment across accounts
- Implement compliance monitoring dashboards

## Common Issues

- symptoms: "Firewall not deployed in target account"
  diagnosis: "Account not in FMS policy scope or compliance not enforced."
  resolution: "Add account to policy scope. Enable auto-remediation."

- symptoms: "FMS policy shows non-compliant"
  diagnosis: "Firewall configuration doesn't match FMS policy requirements."
  resolution: "Check compliance details. Fix firewall configuration or update FMS policy."

- symptoms: "Rule groups not syncing across accounts"
  diagnosis: "FMS policy references rule groups not shared with target accounts."
  resolution: "Share rule groups via RAM or include in FMS policy definition."

## Output Format

```yaml
root_cause: "firewall_manager — <specific_cause>"
evidence:
  - type: fms_policy
    content: "<FMS policy configuration>"
  - type: compliance
    content: "<compliance status per account>"
  - type: scope
    content: "<policy scope (accounts, OUs, regions)>"
severity: MEDIUM
mitigation:
  immediate: "Fix FMS policy scope or compliance issues"
  long_term: "Implement centralized firewall management with FMS"
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

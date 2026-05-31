---
title: "A2 — Firewall Deployment Issues"
description: "Diagnose firewall deployment and provisioning problems"
status: active
severity: HIGH
triggers:
  - "firewall not ready"
  - "deployment failed"
  - "provisioning error"
  - "firewall stuck"
owner: devops-agent
objective: "Identify and resolve firewall deployment issues"
context: "After creation, the firewall goes through provisioning to create endpoints in specified subnets. Provisioning typically takes 5-10 minutes. The firewall must be in READY state before it can inspect traffic. Deployment issues include stuck provisioning, partial deployment, and configuration sync failures."
---

## Phase 1 — Triage

MUST:
- Check firewall status: `aws network-firewall describe-firewall --firewall-name <name> --query 'FirewallStatus'`
- Verify firewall sync state: check ConfigurationSyncStateSummary
- Check endpoint status per AZ in FirewallStatus.SyncStates
- Verify subnets have available IP addresses

SHOULD:
- Check if firewall is in PROVISIONING state (normal, wait 5-10 min)
- Verify no recent changes caused a re-provisioning
- Check CloudTrail for deployment-related events

MAY:
- Check VPC endpoint status for firewall endpoints
- Verify no subnet or VPC issues blocking endpoint creation

## Phase 2 — Remediate

MUST:
- Wait for provisioning to complete (5-10 minutes is normal)
- Verify all AZ endpoints reach IN_SYNC state
- Check that firewall status is READY before configuring routing

SHOULD:
- Monitor firewall status during changes: `aws network-firewall describe-firewall --firewall-name <name>`
- Avoid making multiple changes simultaneously

MAY:
- Set up CloudWatch alarms for firewall status changes
- Implement deployment automation with status checks

## Common Issues

- symptoms: "Firewall stuck in PROVISIONING"
  diagnosis: "Subnet has no available IPs or VPC issue."
  resolution: "Check subnet available IPs. Verify VPC configuration."

- symptoms: "Some AZs show OUT_OF_SYNC"
  diagnosis: "Configuration change propagating to endpoints."
  resolution: "Wait for sync to complete. Check for conflicting changes."

- symptoms: "Firewall READY but not inspecting traffic"
  diagnosis: "Route tables not configured to send traffic to firewall endpoints."
  resolution: "Configure route tables. This is a routing issue, not deployment."

## Output Format

```yaml
root_cause: "deployment — <specific_cause>"
evidence:
  - type: firewall_status
    content: "<firewall provisioning state>"
  - type: sync_state
    content: "<per-AZ sync status>"
  - type: endpoint_status
    content: "<firewall endpoint status>"
severity: HIGH
mitigation:
  immediate: "Wait for provisioning or fix blocking issue"
  long_term: "Implement deployment monitoring and automation"
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

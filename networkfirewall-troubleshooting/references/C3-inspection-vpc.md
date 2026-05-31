---
title: "C3 — Inspection VPC Design Issues"
description: "Diagnose centralized inspection VPC architecture problems"
status: active
severity: HIGH
triggers:
  - "inspection VPC"
  - "centralized firewall"
  - "hub and spoke"
  - "Transit Gateway inspection"
owner: devops-agent
objective: "Identify and resolve inspection VPC design issues"
context: "Centralized inspection VPC architecture uses a dedicated VPC with Network Firewall to inspect traffic from multiple spoke VPCs via Transit Gateway. Design issues include TGW route table configuration, appliance mode, return routing, and cross-AZ traffic handling."
---

## Phase 1 — Triage

MUST:
- Check inspection VPC firewall status: `aws network-firewall describe-firewall --firewall-name <name>`
- Check Transit Gateway route tables: `aws ec2 describe-transit-gateway-route-tables`
- Verify TGW attachments for inspection VPC: `aws ec2 describe-transit-gateway-vpc-attachments`
- Check inspection VPC route tables for return traffic routing

SHOULD:
- Verify TGW appliance mode is enabled on inspection VPC attachment
- Check spoke VPC route tables point to TGW
- Verify firewall endpoints exist in all required AZs

MAY:
- Diagram the complete hub-and-spoke architecture
- Check for blackhole routes in TGW route tables

## Phase 2 — Remediate

MUST:
- Enable appliance mode on TGW attachment: `aws ec2 modify-transit-gateway-vpc-attachment --transit-gateway-attachment-id <id> --options ApplianceModeSupport=enable`
- Configure TGW route tables to send traffic through inspection VPC
- Configure inspection VPC route tables for return traffic

SHOULD:
- Use separate TGW route tables for spoke and inspection VPCs
- Ensure firewall endpoints in all AZs used by TGW attachment

MAY:
- Implement automated TGW route management
- Set up monitoring for TGW and firewall health

## Common Issues

- symptoms: "Traffic not reaching inspection VPC"
  diagnosis: "TGW route table not routing traffic to inspection VPC attachment."
  resolution: "Add routes in spoke TGW route table pointing to inspection VPC attachment."

- symptoms: "Asymmetric routing through inspection VPC"
  diagnosis: "Appliance mode not enabled on TGW attachment."
  resolution: "Enable appliance mode: ApplianceModeSupport=enable on inspection VPC attachment."

- symptoms: "Return traffic not reaching spoke VPCs"
  diagnosis: "Inspection VPC route table missing routes back to spoke VPCs via TGW."
  resolution: "Add routes in inspection VPC route tables for spoke CIDR blocks via TGW."

## Output Format

```yaml
root_cause: "inspection_vpc — <specific_cause>"
evidence:
  - type: tgw_routes
    content: "<TGW route table entries>"
  - type: appliance_mode
    content: "<appliance mode status>"
  - type: inspection_routes
    content: "<inspection VPC route tables>"
severity: HIGH
mitigation:
  immediate: "Fix TGW and inspection VPC routing"
  long_term: "Document centralized inspection architecture"
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

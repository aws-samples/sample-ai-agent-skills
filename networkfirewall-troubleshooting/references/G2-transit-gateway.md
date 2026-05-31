---
title: "G2 — Transit Gateway Integration Issues"
description: "Diagnose Transit Gateway integration problems with Network Firewall"
status: active
severity: HIGH
triggers:
  - "Transit Gateway"
  - "TGW routing"
  - "centralized inspection"
  - "TGW attachment"
owner: devops-agent
objective: "Identify and resolve Transit Gateway integration issues with Network Firewall"
context: "Transit Gateway enables centralized traffic inspection by routing inter-VPC and internet-bound traffic through an inspection VPC with Network Firewall. Key requirements include TGW route tables, appliance mode, and symmetric routing. This is the most common architecture for multi-VPC firewall inspection."
---

## Phase 1 — Triage

MUST:
- Check TGW route tables: `aws ec2 describe-transit-gateway-route-tables`
- Check TGW routes: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <rtb-id> --filters Name=type,Values=static,propagated`
- Verify inspection VPC attachment: `aws ec2 describe-transit-gateway-vpc-attachments --filters Name=vpc-id,Values=<inspection-vpc-id>`
- Check appliance mode: `aws ec2 describe-transit-gateway-vpc-attachments --transit-gateway-attachment-ids <attachment-id> --query 'TransitGatewayVpcAttachments[*].Options'`

SHOULD:
- Verify spoke VPC attachments and route table associations
- Check for blackhole routes in TGW route tables
- Verify inspection VPC route tables for return traffic

MAY:
- Diagram the complete TGW routing architecture
- Check TGW Flow Logs for traffic analysis

## Phase 2 — Remediate

MUST:
- Enable appliance mode on inspection VPC attachment: `aws ec2 modify-transit-gateway-vpc-attachment --transit-gateway-attachment-id <id> --options ApplianceModeSupport=enable`
- Configure TGW route tables: spoke routes → inspection VPC, inspection routes → spoke VPCs
- Configure inspection VPC route tables: TGW → firewall → TGW

SHOULD:
- Use separate TGW route tables for spoke and inspection associations
- Add default route (0.0.0.0/0) in spoke TGW route table pointing to inspection VPC

MAY:
- Implement TGW route table automation
- Set up TGW and firewall monitoring

## Common Issues

- symptoms: "Inter-VPC traffic not being inspected"
  diagnosis: "TGW route table not routing through inspection VPC."
  resolution: "Add routes in spoke TGW route table pointing to inspection VPC attachment."

- symptoms: "Traffic drops in inspection VPC"
  diagnosis: "Appliance mode not enabled causing asymmetric routing."
  resolution: "Enable appliance mode on inspection VPC TGW attachment."

- symptoms: "Return traffic not reaching spoke VPCs"
  diagnosis: "Inspection VPC missing return routes via TGW."
  resolution: "Add spoke CIDR routes in inspection VPC route table pointing to TGW."

## Output Format

```yaml
root_cause: "transit_gateway — <specific_cause>"
evidence:
  - type: tgw_routes
    content: "<TGW route table entries>"
  - type: appliance_mode
    content: "<appliance mode status>"
  - type: attachments
    content: "<TGW VPC attachments>"
severity: HIGH
mitigation:
  immediate: "Fix TGW routing and appliance mode"
  long_term: "Document and automate TGW inspection architecture"
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

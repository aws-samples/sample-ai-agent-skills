---
title: "C2 — Asymmetric Routing Issues"
description: "Diagnose asymmetric routing breaking stateful inspection"
status: active
severity: HIGH
triggers:
  - "asymmetric routing"
  - "stateful dropping traffic"
  - "connection tracking"
  - "return traffic dropped"
owner: devops-agent
objective: "Identify and resolve asymmetric routing issues affecting stateful inspection"
context: "Stateful rules track connections bidirectionally. If request and response traffic traverse different firewall endpoints (or one direction bypasses the firewall), the stateful engine cannot match connection state and drops traffic. This is critical in multi-AZ and Transit Gateway designs."
---

## Phase 1 — Triage

MUST:
- Map the traffic flow path for both request and response
- Verify both directions traverse the SAME firewall endpoint
- Check route tables in all involved subnets (workload, firewall, IGW/TGW)
- Identify if traffic crosses AZ boundaries

SHOULD:
- Check VPC Flow Logs for traffic path analysis
- Verify per-AZ route tables point to local firewall endpoints
- Check for NAT gateway or load balancer in the path affecting routing

MAY:
- Diagram the complete bidirectional traffic flow
- Check if recent routing changes introduced asymmetry

## Phase 2 — Remediate

MUST:
- Ensure per-AZ route tables point to the LOCAL AZ firewall endpoint
- Configure IGW/TGW route tables to route return traffic through the same AZ firewall endpoint
- Verify symmetric routing for all traffic flows

SHOULD:
- Use AZ-specific CIDR routes in IGW route table
- Test with traceroute or VPC Flow Logs to confirm symmetric path

MAY:
- Consider appliance mode on Transit Gateway for symmetric routing
- Implement routing validation automation

## Common Issues

- symptoms: "Connections timing out through firewall"
  diagnosis: "Request goes through AZ-a firewall, response through AZ-b."
  resolution: "Configure per-AZ route tables. Ensure both directions use same endpoint."

- symptoms: "Stateful rules dropping legitimate traffic"
  diagnosis: "Firewall sees only one direction of the flow."
  resolution: "Fix routing to ensure symmetric flow through the same firewall endpoint."

- symptoms: "Works in one AZ but not another"
  diagnosis: "Route table in one AZ correctly configured, other AZ has asymmetric routes."
  resolution: "Verify and fix route tables in all AZs."

## Output Format

```yaml
root_cause: "asymmetric_routing — <specific_cause>"
evidence:
  - type: traffic_path
    content: "<request and response traffic paths>"
  - type: route_tables
    content: "<per-AZ route table entries>"
  - type: firewall_endpoints
    content: "<endpoint IDs per AZ>"
severity: HIGH
mitigation:
  immediate: "Fix route tables for symmetric routing"
  long_term: "Implement per-AZ routing architecture and validation"
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

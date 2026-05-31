---
title: "C1 — Route Table Configuration Issues"
description: "Diagnose route table configuration for traffic inspection"
status: active
severity: HIGH
triggers:
  - "route table"
  - "traffic not inspected"
  - "routing configuration"
  - "firewall bypass"
owner: devops-agent
objective: "Identify and resolve route table configuration issues for Network Firewall"
context: "Traffic must be explicitly routed to firewall endpoints via VPC route tables. This is the #1 cause of 'firewall not working.' Workload subnets need routes to firewall endpoints. Internet gateway route tables need return routes through firewall. Each AZ should have its own route table pointing to the local firewall endpoint."
---

## Phase 1 — Triage

MUST:
- Get firewall endpoint IDs: `aws network-firewall describe-firewall --firewall-name <name> --query 'FirewallStatus.SyncStates'`
- Check workload subnet route tables: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<workload-subnet-id>`
- Verify routes point to firewall endpoints (target type: vpce-*)
- Check IGW/NAT route tables for return traffic routing

SHOULD:
- Verify each AZ has routes to its local firewall endpoint
- Check for missing routes (default route, specific CIDR routes)
- Verify no conflicting more-specific routes bypass the firewall

MAY:
- Diagram the complete traffic flow path
- Check VPC Flow Logs to confirm traffic path

## Phase 2 — Remediate

MUST:
- Add routes in workload subnet route tables: `aws ec2 create-route --route-table-id <rtb-id> --destination-cidr-block 0.0.0.0/0 --vpc-endpoint-id <firewall-endpoint-id>`
- Add return routes in IGW route table for ingress traffic
- Use AZ-specific route tables with local firewall endpoints

SHOULD:
- Route internet-bound traffic: workload → firewall → NAT/IGW
- Route ingress traffic: IGW → firewall → workload
- Document routing architecture

MAY:
- Use AWS Transit Gateway for centralized inspection
- Implement automated route table management

## Common Issues

- symptoms: "Firewall deployed but not inspecting traffic"
  diagnosis: "Route tables not configured to send traffic to firewall endpoints."
  resolution: "Add routes pointing to firewall endpoints in workload subnet route tables."

- symptoms: "Outbound traffic works but inbound doesn't"
  diagnosis: "Missing return routes in IGW route table."
  resolution: "Add routes in IGW route table pointing to firewall endpoints for workload CIDRs."

- symptoms: "Traffic inspected in one AZ but not another"
  diagnosis: "Route table in one AZ missing firewall endpoint route."
  resolution: "Add firewall endpoint route in the missing AZ's route table."

## Output Format

```yaml
root_cause: "route_table — <specific_cause>"
evidence:
  - type: workload_routes
    content: "<workload subnet route table>"
  - type: firewall_endpoints
    content: "<firewall endpoint IDs per AZ>"
  - type: igw_routes
    content: "<IGW route table>"
severity: HIGH
mitigation:
  immediate: "Add missing routes to firewall endpoints"
  long_term: "Document and automate routing architecture"
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

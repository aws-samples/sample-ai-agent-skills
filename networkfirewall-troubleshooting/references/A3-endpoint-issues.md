---
title: "A3 — Firewall Endpoint Issues"
description: "Diagnose firewall endpoint creation and health problems"
status: active
severity: HIGH
triggers:
  - "endpoint not created"
  - "endpoint unhealthy"
  - "firewall endpoint"
  - "GWLBe issue"
owner: devops-agent
objective: "Identify and resolve firewall endpoint issues"
context: "Network Firewall creates Gateway Load Balancer endpoints (GWLBe) in specified subnets. These endpoints are the targets for route table entries. Endpoint issues include creation failures, health problems, and AZ-specific issues. Each endpoint handles traffic for its AZ."
---

## Phase 1 — Triage

MUST:
- Check firewall endpoint status: `aws network-firewall describe-firewall --firewall-name <name> --query 'FirewallStatus.SyncStates'`
- Get endpoint IDs from firewall status (Attachment.EndpointId per AZ)
- Verify endpoints exist: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-ids>`
- Check route tables reference correct endpoint IDs

SHOULD:
- Verify endpoint is in the correct subnet and AZ
- Check if endpoint was recently created or modified
- Verify subnet has available IP addresses

MAY:
- Check VPC Flow Logs for traffic through endpoints
- Verify no NACL blocking traffic to/from endpoints

## Phase 2 — Remediate

MUST:
- Ensure firewall subnets have available IPs
- Verify route tables point to correct endpoint IDs
- Wait for endpoint provisioning if recently created

SHOULD:
- Add firewall to additional AZs: `aws network-firewall associate-subnets --firewall-name <name> --subnet-mappings SubnetId=<new-subnet>`
- Update route tables when endpoint IDs change

MAY:
- Monitor endpoint health with CloudWatch
- Implement automated route table updates

## Common Issues

- symptoms: "Endpoint ID changed after firewall update"
  diagnosis: "Some operations recreate endpoints with new IDs."
  resolution: "Update route tables with new endpoint IDs. Automate this process."

- symptoms: "Traffic not reaching firewall in specific AZ"
  diagnosis: "No endpoint in that AZ or route table pointing to wrong endpoint."
  resolution: "Add firewall subnet in the AZ. Update route table."

- symptoms: "Endpoint shows unhealthy"
  diagnosis: "Firewall configuration issue or provisioning in progress."
  resolution: "Check firewall sync state. Wait for provisioning to complete."

## Output Format

```yaml
root_cause: "endpoint_issue — <specific_cause>"
evidence:
  - type: endpoint_status
    content: "<endpoint health and configuration>"
  - type: route_tables
    content: "<route table entries referencing endpoints>"
  - type: sync_state
    content: "<firewall sync state>"
severity: HIGH
mitigation:
  immediate: "Fix endpoint or routing configuration"
  long_term: "Automate route table updates and monitor endpoint health"
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

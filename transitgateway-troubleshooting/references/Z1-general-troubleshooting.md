---
title: "Z1 — General Transit Gateway Troubleshooting"
description: "Catch-all runbook for Transit Gateway issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "TGW not working"
  - "transit gateway problem"
  - "TGW issue"
  - "something wrong with transit gateway"
owner: devops-agent
objective: "Systematically diagnose unspecified Transit Gateway issues"
context: "This catch-all runbook provides a systematic approach when the specific issue is unclear. It covers the most common TGW problems and guides toward the appropriate specific runbook."
---

## Phase 1 — Triage

MUST:
- Check TGW status: `aws ec2 describe-transit-gateways --transit-gateway-ids <tgw-id>`
- List all attachments and their states: `aws ec2 describe-transit-gateway-attachments --filters Name=transit-gateway-id,Values=<tgw-id>`
- List all route tables: `aws ec2 describe-transit-gateway-route-tables --filters Name=transit-gateway-id,Values=<tgw-id>`
- Search for blackhole routes in each route table: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <tgw-rtb-id> --filters Name=state,Values=blackhole`

SHOULD:
- Check route table associations: `aws ec2 get-transit-gateway-route-table-associations --transit-gateway-route-table-id <tgw-rtb-id>`
- Check route table propagations: `aws ec2 get-transit-gateway-route-table-propagations --transit-gateway-route-table-id <tgw-rtb-id>`
- Review CloudTrail for recent TGW-related events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ec2.amazonaws.com --max-results 20`
- Check CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/TransitGateway --metric-name PacketDropCountBlackhole --dimensions Name=TransitGateway,Value=<tgw-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`

MAY:
- Check VPC route tables for TGW entries
- Verify RAM shares for cross-account TGW
- Check multicast domain configuration

## Phase 2 — Remediate

MUST:
- Address any blackhole routes found
- Fix attachment states that are not "available"
- Correct route table associations and propagations

SHOULD:
- Redirect to the appropriate specific runbook based on findings:
  - Creation issues → A1-A3
  - VPC attachment issues → B1-B3
  - Peering issues → C1-C2
  - VPN issues → D1-D2
  - DX issues → E1-E2
  - Routing issues → F1-F3
  - Multicast issues → G1-G2

MAY:
- Perform a full audit of the TGW configuration
- Document findings for future reference

## Common Issues

- symptoms: "General connectivity failure through TGW"
  diagnosis: "Multiple possible causes — follow systematic triage."
  resolution: "Check attachments, route tables, blackhole routes, and VPC route tables in order."

- symptoms: "Intermittent connectivity"
  diagnosis: "Possible AZ-specific issue, appliance mode, or route flapping."
  resolution: "Check cross-AZ configuration, appliance mode, and route stability."

- symptoms: "New VPC cannot reach existing VPCs through TGW"
  diagnosis: "Missing attachment, association, propagation, or VPC route table entry."
  resolution: "Verify all four: attachment exists, associated with route table, propagation enabled, VPC route table has TGW entry."

## Output Format

```yaml
root_cause: "general_tgw_issue — <specific_cause>"
evidence:
  - type: tgw_status
    content: "<TGW state and configuration>"
  - type: attachments
    content: "<attachment states>"
  - type: route_tables
    content: "<route table summary>"
  - type: blackhole_routes
    content: "<any blackhole routes found>"
severity: MEDIUM
mitigation:
  immediate: "Address the identified issue"
  long_term: "Implement comprehensive TGW monitoring and IaC"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Route tables reveal network topology across accounts
- Peering configurations expose cross-region connectivity
- VPN attachment details contain tunnel configurations
- Classification: INTERNAL - network architecture and routing topology

## Prohibited Actions

- NEVER delete transit gateway route tables with active associations
- NEVER remove VPC or VPN attachments without confirming traffic rerouting
- NEVER modify route propagation settings during business hours without approval
- NEVER change peering configurations without coordinating with peer account owners

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest removing route table associations without review"
  - "NEVER suggest disabling route propagation"
  - "NEVER modify production TGW without maintenance window"

---
title: "A2 — Carrier Gateway Issues"
description: "Diagnose carrier gateway configuration and connectivity problems"
status: active
severity: HIGH
triggers:
  - "carrier gateway"
  - "mobile traffic"
  - "carrier network"
  - "CAGW"
owner: devops-agent
objective: "Resolve carrier gateway creation, configuration, and routing issues"
context: "The carrier gateway enables connectivity between Wavelength Zone instances and the carrier's mobile network. Each VPC can have one carrier gateway. It is required for instances to receive traffic from mobile devices."
---

## Phase 1 — Triage

MUST:
- List carrier gateways: `aws ec2 describe-carrier-gateways`
- Check carrier gateway state: `aws ec2 describe-carrier-gateways --query 'CarrierGateways[*].{Id:CarrierGatewayId,VpcId:VpcId,State:State}'`
- Verify route table has carrier gateway route: `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<wavelength-subnet-id>" --query 'RouteTables[*].Routes'`
- Check carrier IP allocations: `aws ec2 describe-addresses --filters "Name=domain,Values=vpc" --query 'Addresses[?CarrierIp!=null]'`

SHOULD:
- Verify the carrier gateway is attached to the correct VPC
- Check security groups allow inbound traffic from carrier network
- Verify carrier IP is associated with the target instance ENI

MAY:
- Review CloudTrail for carrier gateway configuration changes
- Check for VPC-level restrictions

## Phase 2 — Remediate

MUST:
- If no carrier gateway: create one: `aws ec2 create-carrier-gateway --vpc-id <vpc-id>`
- If missing route: add route to carrier gateway: `aws ec2 create-route --route-table-id <rtb-id> --destination-cidr-block 0.0.0.0/0 --carrier-gateway-id <cagw-id>`
- If no carrier IP: allocate and associate one

SHOULD:
- Ensure only Wavelength subnet route tables point to the carrier gateway
- Configure security groups for expected mobile traffic patterns
- Document carrier gateway configuration

MAY:
- Implement monitoring for carrier gateway traffic metrics
- Set up alerts for carrier gateway state changes

## Common Issues

- symptoms: "Mobile devices cannot reach Wavelength instance"
  diagnosis: "No carrier gateway or missing route in Wavelength subnet route table."
  resolution: "Create carrier gateway and add default route pointing to it."

- symptoms: "Carrier gateway in 'pending' state"
  diagnosis: "Carrier gateway creation in progress or VPC configuration issue."
  resolution: "Wait for creation to complete. Verify VPC has no conflicting gateways."

## Output Format

```yaml
root_cause: "carrier_gateway — <specific_cause>"
evidence:
  - type: gateway_status
    content: "<carrier gateway state>"
  - type: route_table
    content: "<Wavelength subnet routes>"
severity: HIGH
mitigation:
  immediate: "Create or fix carrier gateway configuration"
  long_term: "Implement carrier gateway monitoring and IaC management"
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

- Edge deployment configurations reveal application architecture
- Carrier IP assignments expose public-facing endpoints
- Latency data may reveal user geographic distribution
- Classification: INTERNAL - edge infrastructure topology and carrier network details

## Prohibited Actions

- NEVER terminate EC2 instances in Wavelength zones serving live edge traffic
- NEVER modify carrier gateway configurations during peak usage
- NEVER change VPC subnet associations without confirming application connectivity
- NEVER remove carrier IP addresses assigned to production workloads

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"

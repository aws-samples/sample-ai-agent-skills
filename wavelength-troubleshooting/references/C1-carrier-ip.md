---
title: "C1 — Carrier IP Issues"
description: "Diagnose carrier IP allocation and association problems"
status: active
severity: HIGH
triggers:
  - "carrier IP"
  - "carrier address"
  - "mobile IP"
  - "carrier allocation"
owner: devops-agent
objective: "Resolve carrier IP allocation, association, and routing issues"
context: "Carrier IPs are allocated from the carrier's IP pool and provide reachability from the carrier's mobile network. They are different from standard Elastic IPs and must be allocated with the Wavelength Zone's network border group."
---

## Phase 1 — Triage

MUST:
- List carrier IPs: `aws ec2 describe-addresses --filters "Name=domain,Values=vpc" --query 'Addresses[?CarrierIp!=null].{AllocationId:AllocationId,CarrierIp:CarrierIp,InstanceId:InstanceId,NetworkBorderGroup:NetworkBorderGroup}'`
- Check allocation: `aws ec2 describe-addresses --allocation-ids <alloc-id>`
- Verify association with instance: `aws ec2 describe-addresses --filters "Name=instance-id,Values=<instance-id>"`
- Check carrier gateway exists: `aws ec2 describe-carrier-gateways`

SHOULD:
- Verify the network border group matches the Wavelength Zone
- Check route table has carrier gateway route
- Verify security groups allow traffic on the carrier IP

MAY:
- Check for carrier IP quota limits
- Review CloudTrail for allocation/association events

## Phase 2 — Remediate

MUST:
- If no carrier IP: allocate one: `aws ec2 allocate-address --domain vpc --network-border-group <wavelength-zone-group>`
- If not associated: associate with instance: `aws ec2 associate-address --allocation-id <alloc-id> --instance-id <instance-id>`
- Ensure carrier gateway is created and route table configured

SHOULD:
- Use network border group matching the target Wavelength Zone
- Document carrier IP assignments
- Implement tagging for carrier IP management

MAY:
- Automate carrier IP allocation in launch scripts
- Set up monitoring for carrier IP utilization

## Common Issues

- symptoms: "Cannot allocate carrier IP"
  diagnosis: "Wrong network border group or carrier IP quota reached."
  resolution: "Use the correct network border group for the Wavelength Zone. Request quota increase."

- symptoms: "Carrier IP allocated but instance not reachable"
  diagnosis: "Carrier IP not associated with instance or carrier gateway missing."
  resolution: "Associate carrier IP with instance ENI. Verify carrier gateway and routes."

## Output Format

```yaml
root_cause: "carrier_ip — <specific_cause>"
evidence:
  - type: carrier_ip_status
    content: "<carrier IP allocation and association>"
  - type: carrier_gateway
    content: "<carrier gateway configuration>"
severity: HIGH
mitigation:
  immediate: "Allocate and associate carrier IP correctly"
  long_term: "Automate carrier IP management and monitoring"
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

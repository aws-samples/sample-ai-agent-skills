---
title: "C2 — VPC Subnet Configuration for Wavelength"
description: "Diagnose VPC and subnet configuration issues for Wavelength Zones"
status: active
severity: MEDIUM
triggers:
  - "wavelength subnet"
  - "VPC wavelength"
  - "subnet configuration"
  - "wavelength networking"
owner: devops-agent
objective: "Resolve VPC subnet creation and configuration issues for Wavelength Zones"
context: "Wavelength subnets are created in the same VPC as Region subnets but in a Wavelength Zone AZ. Route tables must point to the carrier gateway for mobile traffic. Security groups and NACLs apply normally."
---

## Phase 1 — Triage

MUST:
- List Wavelength subnets: `aws ec2 describe-subnets --filters "Name=availability-zone,Values=<wavelength-zone>" --query 'Subnets[*].{Id:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,VpcId:VpcId}'`
- Check route table: `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<subnet-id>"`
- Verify carrier gateway route exists in the route table
- Check security groups: `aws ec2 describe-security-groups --filters "Name=vpc-id,Values=<vpc-id>"`

SHOULD:
- Verify CIDR does not overlap with other subnets
- Check NACLs for the Wavelength subnet
- Verify VPC has DNS support enabled

MAY:
- Review VPC flow logs for traffic analysis
- Check for subnet-level resource quotas

## Phase 2 — Remediate

MUST:
- If no Wavelength subnet: create one: `aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block <cidr> --availability-zone <wavelength-zone>`
- If wrong route table: create and associate a route table with carrier gateway route
- Ensure security groups allow required traffic

SHOULD:
- Use separate route tables for Wavelength subnets (carrier gateway) vs Region subnets (IGW/NAT)
- Tag Wavelength subnets for identification
- Document subnet CIDR allocation plan

MAY:
- Implement VPC flow logs for Wavelength subnets
- Set up network monitoring dashboards

## Common Issues

- symptoms: "Cannot create subnet in Wavelength Zone"
  diagnosis: "Wavelength Zone not opted-in or VPC CIDR exhausted."
  resolution: "Opt-in to the zone. Add secondary CIDR to VPC if needed."

- symptoms: "Wavelength instances cannot communicate with Region instances"
  diagnosis: "Route table misconfiguration or security group blocking."
  resolution: "Verify route tables allow VPC-internal routing. Check security groups."

## Output Format

```yaml
root_cause: "vpc_subnet — <specific_cause>"
evidence:
  - type: subnet_config
    content: "<subnet details>"
  - type: route_table
    content: "<route table entries>"
severity: MEDIUM
mitigation:
  immediate: "Fix subnet or route table configuration"
  long_term: "Implement IaC for Wavelength networking"
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

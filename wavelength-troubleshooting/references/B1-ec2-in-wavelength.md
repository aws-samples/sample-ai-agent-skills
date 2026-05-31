---
title: "B1 — EC2 in Wavelength Issues"
description: "Diagnose EC2 instance problems in AWS Wavelength Zones"
status: active
severity: HIGH
triggers:
  - "EC2 wavelength"
  - "instance launch wavelength"
  - "wavelength instance"
  - "compute wavelength"
owner: devops-agent
objective: "Resolve EC2 instance launch, runtime, and networking issues in Wavelength Zones"
context: "EC2 in Wavelength Zones supports limited instance types. Instances use carrier IPs for mobile network reachability. Launch failures may be due to unsupported types, capacity, or subnet configuration."
---

## Phase 1 — Triage

MUST:
- Check instance status: `aws ec2 describe-instances --instance-ids <instance-id>`
- Verify instance type availability: `aws ec2 describe-instance-type-offerings --location-type availability-zone --filters "Name=location,Values=<wavelength-zone>"`
- Check subnet configuration: `aws ec2 describe-subnets --subnet-ids <subnet-id> --query 'Subnets[*].{Id:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}'`
- Verify carrier IP association: `aws ec2 describe-addresses --filters "Name=instance-id,Values=<instance-id>"`

SHOULD:
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Verify route table configuration for the Wavelength subnet
- Check instance system and instance status checks

MAY:
- Review instance console output: `aws ec2 get-console-output --instance-id <instance-id>`
- Check CloudWatch metrics for the instance

## Phase 2 — Remediate

MUST:
- If unsupported type: use a type available in the Wavelength Zone
- If subnet wrong: launch in a Wavelength Zone subnet (AZ type = wavelength-zone)
- If no carrier IP: allocate and associate a carrier IP for mobile reachability

SHOULD:
- Use launch templates with Wavelength-compatible configurations
- Ensure AMI is available in the parent Region
- Configure user data for Wavelength-specific networking

MAY:
- Implement placement strategies for Wavelength instances
- Set up instance recovery automation

## Common Issues

- symptoms: "RunInstances fails with unsupported instance type"
  diagnosis: "Instance type not available in the target Wavelength Zone."
  resolution: "Check available types with describe-instance-type-offerings for the zone."

- symptoms: "Instance running but not reachable from mobile devices"
  diagnosis: "No carrier IP associated or carrier gateway not configured."
  resolution: "Allocate carrier IP and associate with instance. Verify carrier gateway and routes."

## Output Format

```yaml
root_cause: "ec2_wavelength — <specific_cause>"
evidence:
  - type: instance_details
    content: "<instance configuration>"
  - type: carrier_ip
    content: "<carrier IP association>"
severity: HIGH
mitigation:
  immediate: "Fix instance type, networking, or carrier IP"
  long_term: "Implement Wavelength-specific launch templates"
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

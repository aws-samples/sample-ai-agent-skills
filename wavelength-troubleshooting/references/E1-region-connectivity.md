---
title: "E1 — Region Connectivity from Wavelength"
description: "Diagnose connectivity issues between Wavelength Zones and the parent AWS Region"
status: active
severity: HIGH
triggers:
  - "wavelength region"
  - "cannot reach AWS services"
  - "wavelength to region"
  - "VPC routing wavelength"
owner: devops-agent
objective: "Restore connectivity between Wavelength Zone instances and AWS Region services"
context: "Wavelength instances access Region services through VPC routing. Most AWS services (RDS, S3, Lambda, etc.) are in the Region. Proper route table configuration is essential for Wavelength-to-Region communication."
---

## Phase 1 — Triage

MUST:
- Check Wavelength subnet route table: `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<wavelength-subnet-id>"`
- Verify VPC has Region subnets with NAT/IGW: `aws ec2 describe-subnets --filters "Name=vpc-id,Values=<vpc-id>" --query 'Subnets[*].{Id:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}'`
- Check VPC endpoints: `aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=<vpc-id>"`
- Verify DNS resolution: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`

SHOULD:
- Test connectivity from Wavelength instance to Region service endpoints
- Check security groups for outbound rules
- Verify NACLs allow return traffic

MAY:
- Check VPC flow logs for dropped packets
- Review VPC peering or Transit Gateway if multi-VPC

## Phase 2 — Remediate

MUST:
- If no route to Region: ensure VPC routing allows Wavelength-to-Region traffic
- If DNS failing: verify VPC DNS support is enabled
- If security groups blocking: add outbound rules for AWS service endpoints

SHOULD:
- Use VPC gateway endpoints for S3 and DynamoDB
- Use VPC interface endpoints for other frequently accessed services
- Implement proper NACL rules for bidirectional traffic

MAY:
- Set up Transit Gateway for complex multi-VPC architectures
- Implement local caching to reduce Region dependencies

## Common Issues

- symptoms: "Wavelength instance cannot reach S3"
  diagnosis: "No route to Region or no S3 VPC endpoint."
  resolution: "Add S3 gateway endpoint to VPC. Verify route table includes endpoint route."

- symptoms: "DNS resolution fails from Wavelength instance"
  diagnosis: "VPC DNS support disabled or DHCP options misconfigured."
  resolution: "Enable DNS support on VPC. Verify DHCP options set."

## Output Format

```yaml
root_cause: "region_connectivity — <specific_cause>"
evidence:
  - type: route_table
    content: "<Wavelength subnet routes>"
  - type: vpc_endpoints
    content: "<VPC endpoint configuration>"
severity: HIGH
mitigation:
  immediate: "Fix routing or VPC endpoint configuration"
  long_term: "Implement VPC endpoints and minimize Region dependencies"
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

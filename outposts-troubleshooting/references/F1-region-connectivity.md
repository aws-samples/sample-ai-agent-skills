---
title: "F1 — Region Connectivity Issues"
description: "Diagnose connectivity problems between Outpost workloads and AWS Region services"
status: active
severity: HIGH
triggers:
  - "region connectivity"
  - "cannot reach AWS services"
  - "outpost to region"
  - "VPC connectivity outpost"
owner: devops-agent
objective: "Restore connectivity between Outpost workloads and AWS Region services"
context: "Outpost instances access Region services through the VPC (via service link for management, and VPC routing for data). Issues may involve route tables, NAT gateways, VPC endpoints, or DNS resolution."
---

## Phase 1 — Triage

MUST:
- Check Outpost subnet route table: `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<outpost-subnet-id>"`
- Verify NAT gateway or internet gateway: `aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=<vpc-id>"`
- Check VPC endpoints: `aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=<vpc-id>"`
- Verify DNS resolution: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`
- Check service link status: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 60 --statistics Average`

SHOULD:
- Verify security groups allow outbound traffic to Region services
- Check NACLs on Outpost subnets
- Test connectivity from an Outpost instance to Region endpoints

MAY:
- Check for VPC peering or Transit Gateway connectivity
- Review VPC flow logs for dropped traffic

## Phase 2 — Remediate

MUST:
- If missing routes: add routes to NAT gateway or internet gateway for Region access
- If DNS failing: enable DNS support and DNS hostnames on the VPC
- If security groups blocking: add outbound rules for required AWS service endpoints

SHOULD:
- Use VPC endpoints for AWS services to avoid NAT gateway costs and improve latency
- Implement VPC flow logs for network troubleshooting
- Document the network architecture for Outpost-to-Region connectivity

MAY:
- Implement Transit Gateway for complex multi-VPC architectures
- Set up PrivateLink endpoints for frequently accessed services

## Common Issues

- symptoms: "Outpost instances cannot reach S3 or DynamoDB"
  diagnosis: "Missing route to NAT gateway or no VPC endpoint for the service."
  resolution: "Add NAT gateway route or create a VPC gateway endpoint for S3/DynamoDB."

- symptoms: "DNS resolution fails from Outpost instances"
  diagnosis: "VPC DNS support disabled or DHCP options set misconfigured."
  resolution: "Enable DNS support on VPC. Check DHCP options set for correct DNS servers."

## Output Format

```yaml
root_cause: "region_connectivity — <specific_cause>"
evidence:
  - type: route_table
    content: "<subnet route table entries>"
  - type: vpc_endpoints
    content: "<VPC endpoint configuration>"
severity: HIGH
mitigation:
  immediate: "Fix routing or DNS configuration"
  long_term: "Implement VPC endpoints and comprehensive network monitoring"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — connectivity issues prevent access
- Service Disruption Risk: HIGH — region connectivity loss affects Outpost management
- Compliance Impact: HIGH — region connectivity is required for Outpost operations

## Escalation Conditions
- Complete loss of connectivity to the parent AWS Region
- Region connectivity degradation affecting Outpost operations
- DNS resolution failures for regional services
- VPN or Direct Connect link failures
- Region failover required for Outpost management

## Data Sensitivity
- Network connectivity details — INTERNAL
- VPN/Direct Connect configurations — INTERNAL
- DNS configurations — INTERNAL
- Failover procedures — INTERNAL
- Never expose network topology details externally

## Prohibited Actions
- Do NOT modify region connectivity without coordination
- Do NOT bypass encrypted connectivity requirements
- Do NOT ignore region connectivity degradation
- Do NOT change DNS configurations without testing
- Do NOT disable connectivity monitoring
- Do NOT route Outpost management traffic over untrusted networks

## Phase 3 — Rollback
- Restore previous network connectivity configurations
- Re-establish VPN or Direct Connect links
- Restore DNS configurations
- Verify Outpost management operations after recovery
- Restore monitoring for region connectivity
- Document connectivity issues and resolution steps

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

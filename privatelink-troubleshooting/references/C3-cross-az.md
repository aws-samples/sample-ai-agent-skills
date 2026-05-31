---
title: "C3 — Cross-AZ Issues"
description: "Diagnose cross-AZ traffic and cost issues with VPC interface endpoints"
status: active
severity: MEDIUM
triggers:
  - "cross-AZ"
  - "AZ placement"
  - "data transfer cost"
  - "endpoint not in my AZ"
  - "cross availability zone"
owner: devops-agent
objective: "Optimize endpoint AZ placement to minimize cross-AZ costs and latency"
context: "Interface endpoint ENIs are placed in specific subnets/AZs. Traffic from a client in one AZ to an endpoint ENI in another AZ incurs cross-AZ data transfer charges (~$0.01/GB each direction). For cost optimization, place endpoint ENIs in all AZs where clients reside. Some services provide AZ-specific DNS names."
---

## Phase 1 — Triage

MUST:
- Check endpoint subnet/AZ placement: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id> --query 'VpcEndpoints[*].SubnetIds'`
- Check endpoint ENI AZ distribution: `aws ec2 describe-network-interfaces --filters Name=vpc-endpoint-id,Values=<endpoint-id> --query 'NetworkInterfaces[*].{Id:NetworkInterfaceId,SubnetId:SubnetId,AZ:AvailabilityZone,PrivateIp:PrivateIpAddress}'`
- Identify which AZs have clients that use this endpoint
- Check endpoint DNS entries for AZ-specific names: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id> --query 'VpcEndpoints[*].DnsEntries'`

SHOULD:
- Estimate cross-AZ data transfer volume
- Check if clients are using AZ-specific DNS names (if available)
- Verify subnet availability in all required AZs

MAY:
- Analyze VPC Flow Logs for cross-AZ traffic patterns
- Calculate cost impact of cross-AZ traffic

## Phase 2 — Remediate

MUST:
- Add endpoint ENIs to all AZs where clients reside: `aws ec2 modify-vpc-endpoint --vpc-endpoint-id <endpoint-id> --add-subnet-ids <subnet-ids>`
- Verify one subnet per AZ is specified

SHOULD:
- Use AZ-specific DNS names where available to route traffic locally
- Monitor cross-AZ data transfer costs in Cost Explorer
- Place endpoint ENIs in at least 2 AZs for high availability

MAY:
- Implement AZ-aware routing in the application
- Set up cost alerts for cross-AZ data transfer

## Common Issues

- symptoms: "High cross-AZ data transfer costs from PrivateLink"
  diagnosis: "Endpoint ENIs not in all AZs where clients reside."
  resolution: "Add endpoint ENIs to all client AZs. Use AZ-specific DNS names."

- symptoms: "Endpoint only in one AZ — single point of failure"
  diagnosis: "Endpoint created with only one subnet."
  resolution: "Add subnets in additional AZs for high availability."

- symptoms: "Cannot add subnet — already have one in that AZ"
  diagnosis: "Only one subnet per AZ per endpoint is allowed."
  resolution: "Use a different subnet in that AZ, or the existing one is already correct."

## Output Format

```yaml
root_cause: "cross_az — <specific_cause>"
evidence:
  - type: endpoint_azs
    content: "<endpoint ENI AZ distribution>"
  - type: client_azs
    content: "<client AZ distribution>"
  - type: cost_impact
    content: "<estimated cross-AZ data transfer cost>"
severity: MEDIUM
mitigation:
  immediate: "Add endpoint ENIs to all client AZs"
  long_term: "Implement AZ-aware routing and cost monitoring"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — cross-AZ is a networking concern
- Service Disruption Risk: MEDIUM — cross-AZ issues affect endpoint availability
- Compliance Impact: LOW — cross-AZ is an operational concern

## Escalation Conditions
- Cross-AZ connectivity fails
- Cross-AZ latency exceeds thresholds
- AZ failure affects endpoint availability
- Cross-AZ data transfer costs unexpected
- Endpoint not available in required AZs

## Data Sensitivity
- AZ configurations — INTERNAL
- Cross-AZ traffic patterns — INTERNAL
- Latency metrics — INTERNAL
- Cost data — INTERNAL
- Never expose AZ-specific architecture details

## Prohibited Actions
- Do NOT configure endpoints in single AZ for production
- Do NOT ignore cross-AZ latency issues
- Do NOT bypass cross-AZ redundancy requirements
- Do NOT ignore cross-AZ cost implications
- Do NOT modify AZ configurations during production hours without approval
- Do NOT disable cross-AZ monitoring

## Phase 3 — Rollback
- Restore previous AZ configurations
- Re-enable cross-AZ connectivity
- Revert AZ-specific changes
- Verify endpoint availability across AZs
- Document cross-AZ issues and configurations
- Update cross-AZ architecture guidelines

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

---
title: "K2 — Cross-AZ Latency and Cost"
description: "Diagnose performance and cost issues from cross-AZ traffic"
status: active
severity: MEDIUM
triggers:
  - "High latency between services"
  - "Unexpected data transfer costs"
  - "Cross-AZ traffic"
owner: devops-agent
objective: "Identify cross-AZ traffic patterns and optimize placement"
context: "Cross-AZ traffic incurs data transfer charges ($0.01/GB each way in most regions) and adds ~1-2ms latency. Same-AZ traffic is free and lower latency. Multi-AZ is needed for HA but should be optimized."
---

## Phase 1 — Triage

MUST:
- Identify the AZ placement of communicating resources
- Check if high-frequency communication is happening cross-AZ
- Review data transfer costs in Cost Explorer filtered by AZ
- Measure latency between the resources

SHOULD:
- Check if ELB is routing cross-AZ unnecessarily
- Verify if AZ-affinity routing is possible for the workload
- Check NAT gateway placement (cross-AZ NAT adds cost and latency)

MAY:
- Analyze flow logs to quantify cross-AZ traffic volume
- Evaluate if the workload can use AZ-aware routing

## Common Issues

- symptoms: "High data transfer costs, services in different AZs"
  diagnosis: "High-volume traffic between services in different AZs."
  resolution: "Co-locate high-bandwidth services in the same AZ, use AZ-affinity routing."

- symptoms: "All private subnets routing through NAT in a single AZ"
  diagnosis: "Cross-AZ NAT traffic incurs data transfer charges."
  resolution: "Deploy one NAT gateway per AZ, route each AZ's traffic through its own NAT."

## Safety Ratings

```
safety_ratings:
  - "describe-subnets: GREEN — read-only subnet AZ placement check"
  - "describe-instances: GREEN — read-only instance AZ placement check"
  - "describe-nat-gateways: GREEN — read-only NAT gateway AZ check"
  - "get-cost-and-usage (Cost Explorer): GREEN — read-only cost analysis"
  - "describe-flow-logs: GREEN — read-only flow log query for traffic analysis"
  - "create-nat-gateway (add per-AZ NAT): YELLOW — creates NAT gateway, recoverable by deleting"
  - "replace-route (redirect to per-AZ NAT): YELLOW — modifies route, recoverable by replacing back"
```

## Escalation Conditions

- "Fix requires creating additional NAT gateways in production VPC"
- "Fix requires modifying production VPC route tables"
- "Fix requires migrating resources between AZs — service disruption possible"
- "High cross-AZ data transfer costs affecting budget"

## Data Sensitivity

- HIGH: VPC flow logs (contain source/destination IPs and traffic volumes)
- MEDIUM: AZ placement details, data transfer cost information
- MEDIUM: NAT gateway configuration, route table associations

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER consolidate to single AZ to reduce costs (breaks high availability)"

## Phase 3 — Rollback

- If a per-AZ NAT gateway was created: delete it with `aws ec2 delete-nat-gateway --nat-gateway-id <id>` and release the EIP
- If routes were changed to per-AZ NAT: replace them back with `aws ec2 replace-route --route-table-id <rtb-id> --destination-cidr-block 0.0.0.0/0 --nat-gateway-id <original-nat-id>`
- If resources were migrated between AZs: migrate them back (if original placement is still available)
- Document NAT gateway placements, route tables, and resource AZ assignments before changes

## Output Format

```yaml
root_cause: "cross_az_latency — <detail>"
evidence:
  - type: network
    content: "<AZ placement and traffic pattern>"
severity: MEDIUM
mitigation:
  immediate: "Co-locate high-traffic services in same AZ"
  long_term: "Deploy per-AZ NAT, use AZ-affinity routing, monitor cross-AZ costs"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"

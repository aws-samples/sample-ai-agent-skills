---
title: "A3 — Longest Prefix Match Issues"
description: "Diagnose routing issues caused by unexpected longest prefix match behavior"
status: active
severity: MEDIUM
triggers:
  - "Traffic going to wrong destination"
  - "Unexpected routing path"
  - "More specific route overriding"
owner: devops-agent
objective: "Identify when longest prefix match causes traffic to take an unexpected path"
context: "VPC route tables use longest prefix match — the most specific CIDR wins. A /28 route beats a /16 route. This can cause traffic to go to unexpected targets when overlapping CIDRs exist."
---

## Phase 1 — Triage

MUST:
- Get the route table for the affected subnet: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- List all routes and identify overlapping CIDRs
- Determine which route matches the destination IP (most specific CIDR wins)
- Verify the winning route points to the intended target

SHOULD:
- Check if a more specific route was recently added that overrides a broader route
- Use Reachability Analyzer to confirm the actual path taken
- Check flow logs to see where traffic is actually going

MAY:
- Map out all route tables in the VPC to find inconsistencies
- Check if TGW route propagation added an unexpected more-specific route

## Common Issues

- symptoms: "Traffic to 10.1.2.0/24 goes to peering instead of TGW"
  diagnosis: "A /24 route to peering connection is more specific than the /16 route to TGW."
  resolution: "Adjust routes to ensure the correct target for the specific CIDR, or remove the conflicting more-specific route."

- symptoms: "Some IPs in a CIDR work, others don't"
  diagnosis: "A more specific route covers part of the CIDR and sends it to a different (possibly broken) target."
  resolution: "Identify the overlapping more-specific route and fix or remove it."

## Safety Ratings

```
safety_ratings:
  - "describe-route-tables: GREEN — read-only inspection of route entries"
  - "describe-flow-logs: GREEN — read-only flow log query"
  - "search-transit-gateway-routes: GREEN — read-only TGW route inspection"
  - "delete-route (remove conflicting route): YELLOW — removes a route entry, recoverable by re-adding"
  - "create-route (add correct route): YELLOW — adds a new route, recoverable by deleting"
  - "replace-route: YELLOW — modifies route target, recoverable by replacing back"
```

## Escalation Conditions

- "Fix requires modifying production VPC route tables"
- "Fix requires changing Transit Gateway route tables"
- "Multiple subnets or AZs affected by routing conflict"
- "Conflicting route was added by route propagation (TGW/VPN) — may reappear"

## Data Sensitivity

- HIGH: route table entries (expose network architecture and traffic paths)
- HIGH: VPC flow logs (contain source/destination IPs and ports)
- MEDIUM: subnet CIDR ranges and VPC peering configurations

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER remove a more-specific route without understanding what traffic it serves"

## Phase 3 — Rollback

- If a conflicting route was deleted via `delete-route`: re-add it with `aws ec2 create-route --route-table-id <rtb-id> --destination-cidr-block <cidr> --gateway-id <original-target>`
- If a route was replaced via `replace-route`: replace it back with `aws ec2 replace-route --route-table-id <rtb-id> --destination-cidr-block <cidr> --gateway-id <original-target>`
- If a new more-specific route was added: delete it with `aws ec2 delete-route --route-table-id <rtb-id> --destination-cidr-block <cidr>`
- Document all route table entries before making changes to enable full rollback

## Output Format

```yaml
root_cause: "longest_prefix_match — <conflicting routes>"
evidence:
  - type: route_table
    content: "<route table entries showing the conflict>"
severity: MEDIUM
mitigation:
  immediate: "Adjust or remove the conflicting route"
  long_term: "Document routing design, avoid overlapping CIDRs across different targets"
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

---
title: "A4 — Asymmetric Routing"
description: "Diagnose issues caused by traffic taking different paths for request and response"
status: active
severity: MEDIUM
triggers:
  - "Connection resets"
  - "Intermittent connectivity"
  - "Stateful firewall dropping return traffic"
owner: devops-agent
objective: "Identify asymmetric routing and restore symmetric traffic flow"
context: "Asymmetric routing occurs when request and response packets take different paths. This breaks stateful inspection (NACLs are stateless so unaffected, but appliance firewalls and some NAT configurations break). Common with multiple route tables, TGW, or multi-AZ NAT."
---

## Phase 1 — Triage

MUST:
- Identify the full path: source → destination and destination → source
- Check route tables for BOTH directions of traffic flow
- Verify if a stateful middlebox (firewall appliance, NAT) is in the path
- Check if source and destination are in different AZs with different NAT gateways

SHOULD:
- Check flow logs on both source and destination ENIs
- Verify TGW route tables for both directions
- Check if appliance mode is enabled on TGW (required for stateful inspection)

MAY:
- Trace the path using Reachability Analyzer in both directions
- Check if a recent route table change introduced the asymmetry

## Common Issues

- symptoms: "Traffic through TGW to firewall appliance drops intermittently"
  diagnosis: "TGW appliance mode not enabled. Return traffic goes to a different AZ's appliance that has no state for the connection."
  resolution: "Enable appliance mode on the TGW VPC attachment for the firewall VPC."

- symptoms: "Multi-AZ setup with per-AZ NAT, cross-AZ traffic fails"
  diagnosis: "Request goes through AZ-a NAT, response comes back through AZ-b NAT. NAT in AZ-b has no connection state."
  resolution: "Ensure each AZ's route table points to its own AZ's NAT gateway."

## Safety Ratings

```
safety_ratings:
  - "describe-route-tables: GREEN — read-only inspection of route entries in both directions"
  - "describe-transit-gateway-vpc-attachments: GREEN — read-only TGW attachment check"
  - "describe-flow-logs: GREEN — read-only flow log query"
  - "replace-route (fix asymmetric path): YELLOW — modifies route target, recoverable by replacing back"
  - "modify-transit-gateway-vpc-attachment (enable appliance mode): YELLOW — changes TGW attachment setting, recoverable by disabling"
  - "create-route (add symmetric return path): YELLOW — adds route entry, recoverable by deleting"
```

## Escalation Conditions

- "Fix requires modifying production VPC route tables"
- "Fix requires modifying Transit Gateway route tables"
- "Fix requires enabling TGW appliance mode — affects all traffic through the attachment"
- "Multiple subnets or AZs affected by asymmetric routing"

## Data Sensitivity

- HIGH: route table entries (expose network architecture and traffic paths)
- HIGH: VPC flow logs (contain source/destination IPs and ports)
- MEDIUM: TGW attachment configurations and AZ placement details

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER modify route tables without verifying both forward and return paths"

## Phase 3 — Rollback

- If a route was replaced via `replace-route`: replace it back with `aws ec2 replace-route --route-table-id <rtb-id> --destination-cidr-block <cidr> --gateway-id <original-target>`
- If TGW appliance mode was enabled: disable it with `aws ec2 modify-transit-gateway-vpc-attachment --transit-gateway-attachment-id <id> --options ApplianceModeSupport=disable`
- If a new route was added: delete it with `aws ec2 delete-route --route-table-id <rtb-id> --destination-cidr-block <cidr>`
- Document route tables for both directions before making changes

## Output Format

```yaml
root_cause: "asymmetric_routing — <detail>"
evidence:
  - type: route_table
    content: "<forward and return path route entries>"
severity: MEDIUM
mitigation:
  immediate: "Fix route tables to ensure symmetric paths"
  long_term: "Enable TGW appliance mode, use per-AZ routing consistency"
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

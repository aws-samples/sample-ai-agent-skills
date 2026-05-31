---
title: "D2 — Split-Horizon DNS"
description: "Diagnose split-horizon DNS issues with VPC interface endpoints"
status: active
severity: MEDIUM
triggers:
  - "split-horizon DNS"
  - "hybrid DNS"
  - "on-premises DNS"
  - "DNS forwarding"
  - "Route 53 Resolver"
owner: devops-agent
objective: "Resolve split-horizon DNS configurations for PrivateLink in hybrid environments"
context: "Split-horizon DNS provides different DNS responses based on the query source (VPC vs on-premises). Route 53 Resolver endpoints enable DNS forwarding between VPC and on-premises. On-premises clients can use PrivateLink through Direct Connect/VPN with proper DNS configuration. Inbound Resolver endpoints allow on-premises to resolve VPC private DNS."
---

## Phase 1 — Triage

MUST:
- Check Route 53 Resolver configuration: `aws route53resolver list-resolver-endpoints`
- Check Resolver rules: `aws route53resolver list-resolver-rules`
- Verify endpoint private DNS is working within the VPC
- Check if on-premises DNS is forwarding to Route 53 Resolver inbound endpoint

SHOULD:
- Verify Route 53 Resolver inbound endpoint IPs: `aws route53resolver list-resolver-endpoint-ip-addresses --resolver-endpoint-id <endpoint-id>`
- Check on-premises DNS forwarder configuration
- Test DNS resolution from on-premises network

MAY:
- Check Route 53 private hosted zones for conflicts
- Verify Direct Connect or VPN connectivity for DNS traffic

## Phase 2 — Remediate

MUST:
- Create Route 53 Resolver inbound endpoint for on-premises to VPC resolution
- Configure on-premises DNS to forward AWS service domains to Resolver inbound endpoint IPs
- Ensure Direct Connect/VPN allows DNS traffic (UDP/TCP 53)

SHOULD:
- Create Resolver outbound endpoint for VPC to on-premises resolution
- Configure Resolver rules for domain-specific forwarding
- Test DNS resolution from both VPC and on-premises

MAY:
- Implement conditional forwarding for specific domains only
- Set up DNS monitoring for resolution failures

## Common Issues

- symptoms: "On-premises clients cannot resolve AWS service to private IPs"
  diagnosis: "No Route 53 Resolver inbound endpoint or DNS forwarding not configured."
  resolution: "Create Resolver inbound endpoint. Configure on-premises DNS to forward to its IPs."

- symptoms: "VPC DNS resolution works but on-premises doesn't"
  diagnosis: "On-premises DNS not forwarding to Route 53 Resolver."
  resolution: "Configure on-premises DNS forwarder to send AWS domain queries to Resolver inbound IPs."

- symptoms: "DNS forwarding works but connection times out from on-premises"
  diagnosis: "Network path from on-premises to endpoint ENI not established."
  resolution: "Verify Direct Connect/VPN routing allows traffic to endpoint ENI private IPs."

## Output Format

```yaml
root_cause: "split_horizon_dns — <specific_cause>"
evidence:
  - type: resolver_config
    content: "<Route 53 Resolver endpoints and rules>"
  - type: dns_forwarding
    content: "<on-premises DNS forwarding configuration>"
  - type: resolution_test
    content: "<DNS resolution from VPC and on-premises>"
severity: MEDIUM
mitigation:
  immediate: "Configure DNS forwarding and Resolver endpoints"
  long_term: "Implement comprehensive hybrid DNS architecture"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — DNS configuration is infrastructure-level
- Service Disruption Risk: MEDIUM — split-horizon issues cause routing problems
- Compliance Impact: MEDIUM — DNS routing affects private connectivity

## Escalation Conditions
- Split-horizon DNS causes incorrect routing
- DNS resolution returns wrong endpoints
- Split-horizon configuration conflicts
- DNS changes break existing split-horizon setup
- Cross-VPC resolution affected by split-horizon

## Data Sensitivity
- DNS zone configurations — INTERNAL
- Split-horizon rules — INTERNAL
- VPC DNS settings — INTERNAL
- Route 53 configurations — INTERNAL
- Never expose DNS architecture details externally

## Prohibited Actions
- Do NOT modify split-horizon DNS without understanding routing impact
- Do NOT create conflicting DNS zones
- Do NOT bypass DNS validation
- Do NOT ignore DNS resolution inconsistencies
- Do NOT modify split-horizon during production hours without approval
- Do NOT disable split-horizon without alternative routing

## Phase 3 — Rollback
- Restore previous split-horizon DNS configurations
- Revert DNS zone changes
- Restore VPC DNS settings
- Verify DNS resolution after changes
- Document DNS configuration changes
- Update split-horizon DNS procedures

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

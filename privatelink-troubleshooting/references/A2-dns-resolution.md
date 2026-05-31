---
title: "A2 — DNS Resolution"
description: "Diagnose DNS resolution issues with VPC interface endpoints"
status: active
severity: HIGH
triggers:
  - "DNS not resolving"
  - "endpoint DNS"
  - "private IP not returned"
  - "public IP returned"
  - "nslookup shows public"
owner: devops-agent
objective: "Resolve DNS resolution issues so traffic routes through the interface endpoint"
context: "Interface endpoints provide DNS entries. With private DNS enabled, the service's default DNS name resolves to the endpoint's private IPs. Without private DNS, only the endpoint-specific DNS names resolve to private IPs. Private DNS requires enableDnsSupport and enableDnsHostnames on the VPC."
---

## Phase 1 — Triage

MUST:
- Check endpoint DNS entries: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id> --query 'VpcEndpoints[*].DnsEntries'`
- Check if private DNS is enabled: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id> --query 'VpcEndpoints[*].PrivateDnsEnabled'`
- Check VPC DNS attributes:
  - `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`
  - `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsHostnames`
- Verify endpoint state is available: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id> --query 'VpcEndpoints[*].State'`

SHOULD:
- Test DNS resolution from within the VPC: `nslookup <service-dns-name>` from an EC2 instance
- Check if the endpoint has ENIs with private IPs: `aws ec2 describe-network-interfaces --filters Name=vpc-endpoint-id,Values=<endpoint-id>`
- Verify Route 53 Resolver rules are not overriding endpoint DNS

MAY:
- Check for custom DNS configurations (DHCP options set) that may interfere
- Verify no Route 53 private hosted zone overrides the endpoint DNS

## Phase 2 — Remediate

MUST:
- Enable VPC DNS attributes if not set:
  - `aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --enable-dns-support '{"Value":true}'`
  - `aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --enable-dns-hostnames '{"Value":true}'`
- Enable private DNS on the endpoint if needed: `aws ec2 modify-vpc-endpoint --vpc-endpoint-id <endpoint-id> --private-dns-enabled`

SHOULD:
- Use the endpoint-specific DNS name if private DNS cannot be enabled
- Verify DNS resolution after changes from within the VPC

MAY:
- Configure Route 53 Resolver rules for hybrid DNS scenarios
- Use AZ-specific DNS names for optimal routing

## Common Issues

- symptoms: "Service DNS resolves to public IPs"
  diagnosis: "Private DNS not enabled on the endpoint or VPC DNS attributes missing."
  resolution: "Enable private DNS and both VPC DNS attributes (enableDnsSupport, enableDnsHostnames)."

- symptoms: "Endpoint-specific DNS works but service DNS doesn't"
  diagnosis: "Private DNS is not enabled. Only endpoint-specific DNS names resolve to private IPs."
  resolution: "Enable private DNS to override the service's default DNS resolution."

- symptoms: "DNS resolution fails entirely"
  diagnosis: "VPC enableDnsSupport is false or DHCP options set uses custom DNS servers."
  resolution: "Enable enableDnsSupport. If using custom DNS, configure forwarding to VPC DNS."

## Output Format

```yaml
root_cause: "dns_resolution — <specific_cause>"
evidence:
  - type: dns_entries
    content: "<endpoint DNS entries>"
  - type: private_dns
    content: "<private DNS enabled status>"
  - type: vpc_dns
    content: "<VPC DNS attributes>"
severity: HIGH
mitigation:
  immediate: "Enable private DNS and VPC DNS attributes"
  long_term: "Document DNS requirements for all PrivateLink endpoints"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — DNS resolution is infrastructure-level
- Service Disruption Risk: HIGH — resolution failures prevent endpoint access
- Compliance Impact: MEDIUM — DNS resolution affects private connectivity

## Escalation Conditions
- DNS resolution completely fails for endpoints
- Resolution returns incorrect IP addresses
- DNS caching causes stale endpoint addresses
- Cross-VPC DNS resolution fails
- DNS resolution latency affects application performance

## Data Sensitivity
- DNS resolution data — INTERNAL
- Endpoint IP addresses — INTERNAL
- VPC DNS configurations — INTERNAL
- Route 53 resolver settings — INTERNAL
- Never expose internal IP addresses externally

## Prohibited Actions
- Do NOT modify DNS resolver configurations without testing
- Do NOT ignore DNS resolution failures
- Do NOT bypass DNS security features
- Do NOT create DNS records that conflict with endpoint resolution
- Do NOT disable DNS resolution for active endpoints
- Do NOT expose internal DNS records externally

## Phase 3 — Rollback
- Restore previous DNS resolver configurations
- Revert DNS record changes
- Clear DNS caches if stale entries are causing issues
- Verify DNS resolution after changes
- Restore Route 53 resolver settings
- Document DNS resolution issues and configurations

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

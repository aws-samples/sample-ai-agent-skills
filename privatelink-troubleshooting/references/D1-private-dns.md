---
title: "D1 — Private DNS"
description: "Diagnose private DNS configuration issues with VPC interface endpoints"
status: active
severity: HIGH
triggers:
  - "private DNS"
  - "DNS override"
  - "privateDnsEnabled"
  - "service DNS not resolving privately"
  - "enableDnsHostnames"
owner: devops-agent
objective: "Resolve private DNS configuration for transparent PrivateLink usage"
context: "Private DNS enables the service's default DNS name to resolve to the endpoint's private IPs instead of public IPs. This requires enableDnsSupport=true and enableDnsHostnames=true on the VPC, plus privateDnsEnabled=true on the endpoint. Private DNS affects ALL resources in the VPC. For custom endpoint services, domain verification is required."
---

## Phase 1 — Triage

MUST:
- Check private DNS status on endpoint: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id> --query 'VpcEndpoints[*].PrivateDnsEnabled'`
- Check VPC DNS attributes:
  - `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`
  - `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsHostnames`
- Verify DNS resolution from within the VPC (should return private IPs)
- Check endpoint DNS entries: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id> --query 'VpcEndpoints[*].DnsEntries'`

SHOULD:
- Check for Route 53 private hosted zones that may conflict
- Verify DHCP options set is using AmazonProvidedDNS or compatible resolver
- Test DNS resolution from multiple instances in the VPC

MAY:
- Check Route 53 Resolver rules for forwarding conflicts
- Verify private DNS name verification status for custom endpoint services

## Phase 2 — Remediate

MUST:
- Enable VPC DNS attributes:
  - `aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --enable-dns-support '{"Value":true}'`
  - `aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --enable-dns-hostnames '{"Value":true}'`
- Enable private DNS on endpoint: `aws ec2 modify-vpc-endpoint --vpc-endpoint-id <endpoint-id> --private-dns-enabled`

SHOULD:
- Verify DNS resolution changes after enabling private DNS
- Document the impact on all VPC resources (private DNS affects entire VPC)
- Test application connectivity after DNS changes

MAY:
- Configure Route 53 Resolver for hybrid DNS scenarios
- Set up DNS monitoring for resolution changes

## Common Issues

- symptoms: "Private DNS enabled but service DNS still resolves to public IPs"
  diagnosis: "VPC enableDnsHostnames is false."
  resolution: "Enable enableDnsHostnames on the VPC. Both DNS attributes must be true."

- symptoms: "Private DNS breaks other applications in the VPC"
  diagnosis: "Private DNS overrides DNS for ALL resources in the VPC."
  resolution: "This is expected. All VPC resources will use the endpoint. Use endpoint policies to control access."

- symptoms: "Custom endpoint service private DNS verification pending"
  diagnosis: "Domain ownership verification TXT record not added."
  resolution: "Add the TXT record to your DNS zone. Check verification status in endpoint service config."

## Output Format

```yaml
root_cause: "private_dns — <specific_cause>"
evidence:
  - type: private_dns_enabled
    content: "<privateDnsEnabled status>"
  - type: vpc_dns_attributes
    content: "<enableDnsSupport and enableDnsHostnames>"
  - type: dns_resolution
    content: "<actual DNS resolution results>"
severity: HIGH
mitigation:
  immediate: "Enable VPC DNS attributes and private DNS on endpoint"
  long_term: "Document DNS impact and monitor resolution"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — DNS configuration is infrastructure-level
- Service Disruption Risk: HIGH — DNS issues can break all endpoint connectivity
- Compliance Impact: MEDIUM — private DNS enforces private connectivity requirements

## Escalation Conditions
- Private DNS resolution fails for endpoint
- DNS changes cause endpoint connectivity issues
- Private hosted zone association fails
- DNS propagation delays affect service access
- DNS configuration conflicts between endpoints

## Data Sensitivity
- DNS configurations — INTERNAL
- Private hosted zone details — INTERNAL
- DNS record entries — INTERNAL
- VPC DNS settings — INTERNAL
- Never expose internal DNS configurations externally

## Prohibited Actions
- Do NOT modify private DNS without understanding endpoint impact
- Do NOT disable private DNS on endpoints without alternative resolution
- Do NOT create conflicting DNS records
- Do NOT bypass DNS validation
- Do NOT modify VPC DNS settings without coordination
- Do NOT ignore DNS resolution failures

## Phase 3 — Rollback
- Restore previous DNS configurations
- Re-enable private DNS if disabled
- Revert hosted zone changes
- Verify DNS resolution after changes
- Restore VPC DNS settings
- Document DNS changes and their impact

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

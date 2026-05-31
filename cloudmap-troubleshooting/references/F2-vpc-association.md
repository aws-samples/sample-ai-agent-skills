---
title: "F2 — Cloud Map VPC Association Issues"
description: "Diagnose VPC association problems for private DNS namespaces"
status: active
severity: HIGH
triggers:
  - "VPC association"
  - "private namespace VPC"
  - "DNS not working in VPC"
  - "cross-VPC discovery"
owner: devops-agent
objective: "Identify and resolve VPC association issues for private DNS namespaces"
context: "Private DNS namespaces create Route 53 private hosted zones that must be associated with VPCs where clients need DNS resolution. Without association, DNS queries fail."
---

## Phase 1 — Triage

MUST:
- Check namespace type is DNS_PRIVATE: `aws servicediscovery get-namespace --id <namespace-id> --query 'Namespace.Type'`
- Get hosted zone ID: `aws servicediscovery get-namespace --id <namespace-id> --query 'Namespace.Properties.DnsProperties.HostedZoneId'`
- Check VPC associations: `aws route53 get-hosted-zone --id <zone-id> --query 'VPCs'`
- Verify client VPC is associated
- Check VPC DNS settings: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`

SHOULD:
- Verify enableDnsHostnames is also enabled: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsHostnames`
- Check DHCP options set: `aws ec2 describe-dhcp-options --dhcp-options-ids <dhcp-id>`
- Test DNS resolution from within the VPC

MAY:
- Check for conflicting hosted zones with the same domain name
- Verify Route 53 Resolver rules if using hybrid DNS

## Phase 2 — Remediate

MUST:
- Associate VPC with the private hosted zone: `aws route53 associate-vpc-with-hosted-zone --hosted-zone-id <zone-id> --vpc VPCRegion=<region>,VPCId=<vpc-id>`
- Enable DNS support on the VPC: `aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --enable-dns-support`
- Enable DNS hostnames on the VPC: `aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --enable-dns-hostnames`

SHOULD:
- Use AmazonProvidedDNS in DHCP options
- Document which VPCs are associated with which namespaces
- Verify DNS resolution after association

MAY:
- Set up Route 53 Resolver for cross-VPC or hybrid DNS
- Implement VPC association automation

## Common Issues

- symptoms: "DNS resolution fails from a specific VPC"
  diagnosis: "Private hosted zone not associated with that VPC."
  resolution: "Associate the hosted zone with the VPC."

- symptoms: "DNS resolution fails despite VPC association"
  diagnosis: "VPC DNS support or DNS hostnames disabled."
  resolution: "Enable both enableDnsSupport and enableDnsHostnames on the VPC."

- symptoms: "Cross-account VPC cannot resolve Cloud Map DNS"
  diagnosis: "Cross-account VPC association requires authorization."
  resolution: "Create authorization and then associate the cross-account VPC."

## Output Format

```yaml
root_cause: "vpc_association_issue — <specific_cause>"
evidence:
  - type: vpc_associations
    content: "<associated VPCs>"
  - type: vpc_dns_settings
    content: "<DNS support and hostnames status>"
  - type: dns_test
    content: "<DNS resolution test results>"
severity: HIGH
mitigation:
  immediate: "Associate VPC and enable DNS settings"
  long_term: "Automate VPC association in infrastructure provisioning"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Associate VPC with private hosted zone | YELLOW |
| Enable DNS support on VPC | GREEN |
| Enable DNS hostnames on VPC | GREEN |
| Use AmazonProvidedDNS in DHCP options | GREEN |
| Document VPC associations | GREEN |
| Set up Route 53 Resolver for cross-VPC DNS | YELLOW |

## Escalation Conditions

- Fix requires modifying production VPC configurations
- Fix involves associating production VPCs with hosted zones

## Data Sensitivity

- **MEDIUM**: VPC association configurations expose network topology and DNS resolution paths
- Cross-account VPC associations reveal organizational infrastructure relationships

## Prohibited Actions

- NEVER suggest deleting namespaces with active service registrations

## Phase 3 — Rollback

1. If VPC was associated incorrectly, disassociate: `aws route53 disassociate-vpc-from-hosted-zone --hosted-zone-id <zone-id> --vpc VPCRegion=<region>,VPCId=<vpc-id>`
2. If DNS settings were changed on VPC, restore previous enableDnsSupport and enableDnsHostnames values
3. If DHCP options were modified, restore previous DHCP options set
4. Verify rollback by confirming VPC associations and DNS resolution match expected state

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

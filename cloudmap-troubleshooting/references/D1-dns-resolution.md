---
title: "D1 — Cloud Map DNS Resolution Issues"
description: "Diagnose DNS resolution failures for Cloud Map services"
status: active
severity: HIGH
triggers:
  - "DNS not resolving"
  - "cannot resolve"
  - "NXDOMAIN"
  - "DNS lookup failed"
owner: devops-agent
objective: "Identify and resolve DNS resolution issues for Cloud Map DNS namespaces"
context: "DNS resolution for Cloud Map requires DNS namespaces (not HTTP), correct Route 53 records, VPC association for private namespaces, and proper DNS configuration on the client."
---

## Phase 1 — Triage

MUST:
- Verify namespace is DNS type (not HTTP): `aws servicediscovery get-namespace --id <namespace-id> --query 'Namespace.Type'`
- Check Route 53 hosted zone: `aws route53 list-hosted-zones --query 'HostedZones[?Name==\`<namespace-name>.\`]'`
- Check DNS records exist: `aws route53 list-resource-record-sets --hosted-zone-id <zone-id> --query 'ResourceRecordSets[?Name==\`<service>.<namespace>.\`]'`
- For private namespaces, check VPC association: `aws route53 get-hosted-zone --id <zone-id> --query 'VPCs'`
- Verify instances are registered: `aws servicediscovery list-instances --service-id <service-id>`

SHOULD:
- Test DNS resolution from within the VPC: `dig <service>.<namespace>` or `nslookup <service>.<namespace>`
- Check VPC DNS settings: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`
- Verify DHCP options set includes AmazonProvidedDNS

MAY:
- Check Route 53 resolver query logs
- Verify no conflicting DNS records in other hosted zones

## Phase 2 — Remediate

MUST:
- Ensure namespace is DNS type for DNS resolution
- Associate private hosted zone with client VPC
- Enable DNS support and DNS hostnames on the VPC
- Register instances with correct IP attributes

SHOULD:
- Use AmazonProvidedDNS in DHCP options
- Lower TTL during deployments for faster updates
- Verify DNS resolution from multiple clients

MAY:
- Set up Route 53 Resolver for hybrid DNS
- Implement DNS monitoring

## Common Issues

- symptoms: "NXDOMAIN for Cloud Map service hostname"
  diagnosis: "Private hosted zone not associated with the client VPC."
  resolution: "Associate the hosted zone with the VPC."

- symptoms: "DNS resolves but returns no records"
  diagnosis: "No healthy instances registered or all instances unhealthy."
  resolution: "Register instances and verify health check status."

- symptoms: "DNS resolves to stale IP"
  diagnosis: "DNS TTL caching old records after instance deregistration."
  resolution: "Wait for TTL expiration or lower TTL for faster updates."

## Output Format

```yaml
root_cause: "dns_resolution_issue — <specific_cause>"
evidence:
  - type: namespace_type
    content: "<DNS or HTTP>"
  - type: dns_records
    content: "<Route 53 record sets>"
  - type: vpc_association
    content: "<VPC association status>"
severity: HIGH
mitigation:
  immediate: "Fix VPC association or DNS configuration"
  long_term: "Implement DNS monitoring and alerting"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Associate private hosted zone with client VPC | YELLOW |
| Enable DNS support and DNS hostnames on VPC | GREEN |
| Register instances with correct IP attributes | GREEN |
| Use AmazonProvidedDNS in DHCP options | GREEN |
| Lower TTL during deployments | GREEN |
| Set up Route 53 Resolver for hybrid DNS | YELLOW |

## Escalation Conditions

- Fix requires modifying production VPC DNS settings
- Fix involves associating hosted zones with production VPCs

## Data Sensitivity

- **MEDIUM**: DNS resolution configurations expose service discovery topology
- VPC associations and DNS settings reveal network architecture

## Prohibited Actions

- NEVER suggest deleting namespaces with active service registrations

## Phase 3 — Rollback

1. If VPC association was added incorrectly, disassociate the VPC from the hosted zone
2. If DNS settings were changed on VPC, restore previous enableDnsSupport and enableDnsHostnames values
3. If DHCP options were modified, restore previous DHCP options set
4. Verify rollback by confirming DNS resolution behavior matches expected state

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

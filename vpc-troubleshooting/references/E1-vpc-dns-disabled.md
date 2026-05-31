---
title: "E1 — VPC DNS Disabled"
description: "Diagnose DNS resolution failures caused by disabled VPC DNS settings"
status: active
severity: CRITICAL
triggers:
  - "DNS resolution fails"
  - "Can't resolve hostnames"
  - "enableDnsSupport false"
owner: devops-agent
objective: "Restore DNS resolution in the VPC"
context: "enableDnsSupport=true provides the Amazon-provided DNS server at VPC CIDR+2 (e.g., 10.0.0.2). enableDnsHostnames=true assigns DNS hostnames to instances and is required for private hosted zones."
---

## Phase 1 — Triage

MUST:
- Check VPC DNS settings: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`
- Check DNS hostnames: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsHostnames`
- Verify instances can reach the VPC DNS server (CIDR+2, e.g., 10.0.0.2)
- Check security group outbound allows UDP/TCP 53

SHOULD:
- Check if DHCP options set overrides the DNS server
- Verify NACL allows DNS traffic (UDP/TCP 53) to/from VPC CIDR+2
- Check if private hosted zones are associated with the VPC

MAY:
- Test DNS resolution from an instance: `nslookup example.com 10.0.0.2`
- Check Route 53 Resolver rules if custom DNS forwarding is configured

## Common Issues

- symptoms: "All DNS resolution fails in VPC"
  diagnosis: "enableDnsSupport is false. No Amazon-provided DNS server."
  resolution: "`aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --enable-dns-support`"

- symptoms: "Private hosted zone records not resolving"
  diagnosis: "enableDnsHostnames must be true for private hosted zones to work."
  resolution: "Enable both enableDnsSupport and enableDnsHostnames."

## Safety Ratings

```
safety_ratings:
  - "describe-vpc-attribute (enableDnsSupport): GREEN — read-only VPC attribute check"
  - "describe-vpc-attribute (enableDnsHostnames): GREEN — read-only VPC attribute check"
  - "describe-security-groups: GREEN — read-only SG rule inspection"
  - "modify-vpc-attribute (enable DNS support): YELLOW — changes VPC DNS setting, recoverable by disabling"
  - "modify-vpc-attribute (enable DNS hostnames): YELLOW — changes VPC DNS setting, recoverable by disabling"
```

## Escalation Conditions

- "Fix requires modifying VPC DNS attributes in production VPC"
- "All DNS resolution fails in VPC — CRITICAL impact on all services"
- "Fix affects private hosted zone resolution across multiple services"
- "DHCP options set override may need modification"

## Data Sensitivity

- HIGH: VPC DNS configuration (exposes internal DNS architecture)
- HIGH: VPC flow logs (contain source/destination IPs and ports)
- MEDIUM: DHCP options set, private hosted zone associations

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER disable enableDnsSupport without confirming no services depend on VPC DNS"

## Phase 3 — Rollback

- If enableDnsSupport was enabled: disable it with `aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --no-enable-dns-support` (WARNING: breaks all VPC DNS resolution)
- If enableDnsHostnames was enabled: disable it with `aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --no-enable-dns-hostnames`
- Document original VPC DNS attribute values before making changes

## Output Format

```yaml
root_cause: "vpc_dns_disabled — <attribute>"
evidence:
  - type: vpc_attribute
    content: "<DNS support/hostname settings>"
severity: CRITICAL
mitigation:
  immediate: "Enable the required DNS attribute"
  long_term: "Enforce DNS settings via SCP or IaC"
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

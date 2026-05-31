---
title: "C5 — DNS Resolution Failures"
description: "Diagnose DNS resolution failures for EC2 instances in VPC"
status: active
severity: MEDIUM
triggers:
  - "could not resolve host"
  - "DNS.*timeout"
  - "Name or service not known"
  - "NXDOMAIN"
owner: devops-agent
objective: "Identify the DNS resolution issue and restore name resolution"
context: "VPC DNS uses the Amazon-provided DNS server at VPC CIDR base +2 (e.g., 10.0.0.2). Requires enableDnsSupport=true on the VPC. Private hosted zones require enableDnsHostnames=true. Custom DHCP option sets can override the DNS server."
---

## Phase 1 — Triage

MUST:
- Check VPC DNS settings: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`
- Check VPC DNS hostnames: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsHostnames`
- Check DHCP options set: `aws ec2 describe-dhcp-options --dhcp-options-ids <id>` — look for domain-name-servers
- Verify security group allows outbound UDP/TCP port 53

SHOULD:
- Check if NACL blocks UDP/TCP port 53 outbound or return traffic
- If using Route 53 private hosted zones: verify the hosted zone is associated with the VPC
- Check /etc/resolv.conf inside the instance (via SSM) for correct nameserver

MAY:
- Test with `dig @169.254.169.253 <hostname>` (VPC DNS resolver) via SSM
- Check Route 53 Resolver rules if using hybrid DNS

## Common Issues

- symptoms: "All DNS resolution fails"
  diagnosis: "enableDnsSupport is false on the VPC, or DHCP options set points to unreachable DNS server."
  resolution: "Enable DNS support on VPC, or fix DHCP options set."

- symptoms: "Private hosted zone names don't resolve"
  diagnosis: "enableDnsHostnames is false, or hosted zone not associated with the VPC."
  resolution: "Enable DNS hostnames on VPC. Associate the private hosted zone with the VPC."

- symptoms: "DNS works for public names but not private"
  diagnosis: "Route 53 private hosted zone not associated with VPC, or resolver rule missing."
  resolution: "Associate hosted zone with VPC. Check Route 53 Resolver rules."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-vpc-attribute, describe-dhcp-options: GREEN — read-only"
  - "describe-hosted-zones, list-resolver-rules: GREEN — read-only"
  - "Enable DNS support on VPC: YELLOW — affects all instances in VPC, recoverable"
  - "Enable DNS hostnames on VPC: YELLOW — affects all instances in VPC, recoverable"
  - "Associate private hosted zone with VPC: YELLOW — changes DNS resolution, recoverable by disassociating"
  - "Modify DHCP options set: RED — requires new DHCP options set (immutable), affects all instances in VPC"
```

## Escalation Conditions
- VPC DNS settings change would affect all instances in a production VPC
- DHCP options set needs modification (requires creating new set and associating, cannot edit in place)
- DNS resolution failure affects cross-account private hosted zones
- Route 53 Resolver rules need modification affecting hybrid DNS (on-premises integration)
- Multiple VPCs are affected by the same DNS resolution issue

## Data Sensitivity
- HIGH: describe-instances (reveals instance IPs, hostnames, VPC topology)
- MEDIUM: describe-dhcp-options (reveals DNS server configuration, domain names)
- MEDIUM: describe-hosted-zones (reveals internal DNS zone names and VPC associations)
- LOW: describe-vpc-attribute (reveals DNS configuration flags)

## Prohibited Actions
- NEVER suggest disabling DNS support on a VPC to troubleshoot (breaks all DNS resolution)
- NEVER suggest pointing DHCP options to external DNS servers without understanding security implications
- NEVER suggest modifying /etc/resolv.conf directly — it will be overwritten by DHCP
- NEVER suggest associating a private hosted zone with a VPC in another account without proper authorization

## Phase 3 — Rollback
- If DNS support was enabled: disable with `modify-vpc-attribute --no-enable-dns-support` (caution: breaks DNS)
- If DNS hostnames was enabled: disable with `modify-vpc-attribute --no-enable-dns-hostnames`
- If hosted zone was associated: disassociate with `disassociate-vpc-from-hosted-zone`
- If DHCP options set was changed: associate previous DHCP options set with `associate-dhcp-options`
- If resolver rule was created: delete with `delete-resolver-rule` after disassociating from VPCs

## Output Format

```yaml
root_cause: "<dns_support_disabled|dhcp_misconfigured|hosted_zone_not_associated|sg_blocking_dns>"
evidence:
  - type: vpc_attribute
    content: "<DNS settings from VPC>"
severity: MEDIUM
mitigation:
  immediate: "Enable DNS support/hostnames, fix DHCP options"
  long_term: "Use Route 53 Resolver for hybrid DNS, monitor DNS metrics"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"

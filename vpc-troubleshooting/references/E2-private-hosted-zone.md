---
title: "E2 — Private Hosted Zone Issues"
description: "Diagnose DNS resolution failures with Route 53 private hosted zones"
status: active
severity: HIGH
triggers:
  - "Private DNS not resolving"
  - "Hosted zone records not found"
  - "NXDOMAIN for internal names"
owner: devops-agent
objective: "Fix private hosted zone DNS resolution"
context: "Private hosted zones require: enableDnsSupport=true, enableDnsHostnames=true, and the hosted zone must be associated with the VPC. Records are only resolvable from within associated VPCs."
---

## Phase 1 — Triage

MUST:
- Verify VPC DNS settings (both enableDnsSupport and enableDnsHostnames must be true)
- Check hosted zone is associated with the VPC: `aws route53 get-hosted-zone --id <zone-id>` → VPCs
- Verify the record exists: `aws route53 list-resource-record-sets --hosted-zone-id <zone-id>`
- Test resolution from within the VPC

SHOULD:
- Check if there's a conflicting public hosted zone for the same domain
- Verify no Route 53 Resolver rules override the private zone
- Check if the VPC association was recently removed

MAY:
- Check if split-horizon DNS is intended (same domain in public and private zones)
- Verify cross-account VPC associations if applicable

## Common Issues

- symptoms: "Private zone records resolve from VPC-A but not VPC-B"
  diagnosis: "Private hosted zone not associated with VPC-B."
  resolution: "Associate the hosted zone with VPC-B."

- symptoms: "enableDnsHostnames is false, private zone not working"
  diagnosis: "Private hosted zones require enableDnsHostnames=true."
  resolution: "Enable enableDnsHostnames on the VPC."

## Safety Ratings

```
safety_ratings:
  - "get-hosted-zone: GREEN — read-only hosted zone inspection"
  - "list-resource-record-sets: GREEN — read-only DNS record listing"
  - "describe-vpc-attribute: GREEN — read-only VPC attribute check"
  - "associate-vpc-with-hosted-zone: YELLOW — associates VPC with hosted zone, recoverable by disassociating"
  - "modify-vpc-attribute (enable DNS hostnames): YELLOW — changes VPC setting, recoverable by disabling"
  - "change-resource-record-sets (add/modify records): YELLOW — modifies DNS records, recoverable by reverting"
```

## Escalation Conditions

- "Fix requires associating production VPC with hosted zone"
- "Fix requires modifying VPC DNS attributes in production"
- "Multiple VPCs affected by hosted zone association issues"
- "Fix involves cross-account VPC associations"

## Data Sensitivity

- HIGH: private hosted zone records (expose internal service names and IPs)
- HIGH: VPC associations (expose which VPCs access internal DNS)
- MEDIUM: VPC DNS attribute settings, hosted zone configuration

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER delete a private hosted zone without confirming no services depend on it"

## Phase 3 — Rollback

- If a VPC was associated with a hosted zone: disassociate it with `aws route53 disassociate-vpc-from-hosted-zone --hosted-zone-id <zone-id> --vpc VPCRegion=<region>,VPCId=<vpc-id>`
- If enableDnsHostnames was enabled: disable it with `aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --no-enable-dns-hostnames`
- If DNS records were added/modified: revert with `aws route53 change-resource-record-sets` using DELETE or the original record values
- Document hosted zone associations and VPC DNS attributes before changes

## Output Format

```yaml
root_cause: "private_hosted_zone — <detail>"
evidence:
  - type: route53
    content: "<hosted zone association and VPC DNS settings>"
severity: HIGH
mitigation:
  immediate: "Associate hosted zone with VPC and enable DNS attributes"
  long_term: "Use IaC to manage hosted zone associations"
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

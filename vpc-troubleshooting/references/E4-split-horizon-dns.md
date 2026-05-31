---
title: "E4 — Split-Horizon DNS Issues"
description: "Diagnose DNS resolution issues when the same domain has public and private hosted zones"
status: active
severity: MEDIUM
triggers:
  - "Wrong IP returned for domain"
  - "Internal service resolving to public IP"
  - "Split-horizon DNS mismatch"
owner: devops-agent
objective: "Fix split-horizon DNS configuration so internal and external resolution work correctly"
context: "Split-horizon DNS uses the same domain name in both public and private hosted zones. From within the VPC, the private zone takes precedence. From outside, the public zone is used. Issues arise when records are missing from one zone."
---

## Phase 1 — Triage

MUST:
- List both public and private hosted zones for the domain
- Compare records in both zones
- Verify which zone is being used from the failing client's perspective
- Check if the private zone is associated with the correct VPC

SHOULD:
- Test resolution from inside the VPC (should get private zone answer)
- Test resolution from outside (should get public zone answer)
- Check if a record exists in public but is missing from private zone

MAY:
- Check Route 53 Resolver rules that might override zone selection
- Verify DNSSEC settings aren't interfering

## Common Issues

- symptoms: "Application in VPC resolves api.example.com to public IP instead of private"
  diagnosis: "Private hosted zone missing the record, or not associated with the VPC."
  resolution: "Add the record to the private hosted zone and verify VPC association."

- symptoms: "New record added to public zone but not resolving internally"
  diagnosis: "Private zone takes precedence. Record must be added to private zone too."
  resolution: "Add the record to both public and private hosted zones."

## Safety Ratings

```
safety_ratings:
  - "list-hosted-zones: GREEN — read-only hosted zone listing"
  - "list-resource-record-sets: GREEN — read-only DNS record listing"
  - "describe-vpc-attribute: GREEN — read-only VPC attribute check"
  - "change-resource-record-sets (add missing record): YELLOW — adds DNS record, recoverable by deleting"
  - "associate-vpc-with-hosted-zone: YELLOW — associates VPC with zone, recoverable by disassociating"
```

## Escalation Conditions

- "Fix requires modifying DNS records in production hosted zones"
- "Fix requires adding records to both public and private zones"
- "Internal services resolving to public IPs — potential security exposure"
- "Multiple services affected by split-horizon DNS mismatch"

## Data Sensitivity

- HIGH: private hosted zone records (expose internal service names and IPs)
- HIGH: public hosted zone records (expose external-facing service endpoints)
- MEDIUM: VPC associations, DNS resolution patterns

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER delete records from public hosted zone without confirming external impact"

## Phase 3 — Rollback

- If a DNS record was added to private zone: delete it with `aws route53 change-resource-record-sets` using DELETE action
- If a DNS record was added to public zone: delete it with `aws route53 change-resource-record-sets` using DELETE action
- If a VPC was associated with a hosted zone: disassociate it with `aws route53 disassociate-vpc-from-hosted-zone --hosted-zone-id <zone-id> --vpc VPCRegion=<region>,VPCId=<vpc-id>`
- Document all records in both public and private zones before making changes

## Output Format

```yaml
root_cause: "split_horizon_dns — <detail>"
evidence:
  - type: route53
    content: "<public and private zone records comparison>"
severity: MEDIUM
mitigation:
  immediate: "Add missing records to the appropriate hosted zone"
  long_term: "Automate record sync between public and private zones"
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

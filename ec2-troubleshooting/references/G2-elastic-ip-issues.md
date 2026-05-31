---
title: "G2 — Elastic IP Issues"
description: "Diagnose Elastic IP allocation, association, and connectivity issues"
status: active
severity: MEDIUM
triggers:
  - "AddressLimitExceeded"
  - "EIP.*not associated"
  - "public IP.*changed"
owner: devops-agent
objective: "Resolve EIP issues and restore stable public IP connectivity"
context: "EIPs provide static public IPv4 addresses. Free when associated with a running instance. Charged when unassociated or associated with a stopped instance. Default limit: 5 per region. EIPs in VPC can be moved between instances."
---

## Phase 1 — Triage

MUST:
- List EIPs: `aws ec2 describe-addresses`
- Check if EIP is associated: look for AssociationId and InstanceId
- Verify the instance is running (EIP on stopped instance still costs money)

SHOULD:
- Check for unassociated EIPs (wasting money)
- Verify EIP quota: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-0263D0A3`

## Common Issues

- symptoms: "Public IP changed after stop+start"
  diagnosis: "Instance was using auto-assigned public IP, not an EIP. Auto-assigned IPs change on stop+start."
  resolution: "Allocate and associate an EIP for a static public IP."

- symptoms: "AddressLimitExceeded"
  diagnosis: "EIP quota (default 5 per region) reached."
  resolution: "Release unused EIPs or request quota increase."

- symptoms: "EIP associated but instance unreachable from internet"
  diagnosis: "Missing IGW route, security group blocking, or NACL blocking."
  resolution: "Check route table has 0.0.0.0/0 → IGW. Check SG and NACL rules."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-addresses, describe-instances: GREEN — read-only"
  - "Allocate new EIP: GREEN — creates new resource, no impact"
  - "Associate EIP with instance: YELLOW — changes public IP, recoverable by disassociating"
  - "Disassociate EIP: YELLOW — removes static public IP, instance loses public access"
  - "Release EIP: RED — permanently releases the IP address, cannot reclaim same IP"
```

## Escalation Conditions
- EIP is associated with a production instance and reassociation would cause downtime
- EIP quota is exhausted and no unused EIPs can be released
- EIP is referenced in DNS records, firewall rules, or third-party whitelists
- Multiple EIPs need to be moved during a maintenance window
- EIP belongs to a BYOIP address range with special routing

## Data Sensitivity
- HIGH: describe-addresses (reveals all public IPs, instance associations, allocation IDs)
- MEDIUM: describe-instances (reveals instance public/private IPs)
- LOW: service-quotas (reveals EIP quota limits)

## Prohibited Actions
- NEVER suggest releasing an EIP that is referenced in DNS records without updating DNS first
- NEVER suggest releasing an EIP to free quota without confirming it's truly unused
- NEVER suggest associating an EIP with a stopped instance without explaining the cost
- NEVER suggest using auto-assigned public IPs for services that need stable IP addresses

## Phase 3 — Rollback
- If EIP was associated with wrong instance: disassociate and re-associate with correct instance
- If EIP was disassociated: re-associate with `associate-address`
- If EIP was released: allocate a new EIP (note: you cannot reclaim the same IP address)
- If DNS was updated for new EIP: revert DNS records to previous IP

## Output Format

```yaml
root_cause: "<quota_exceeded|not_associated|missing_igw_route|auto_assigned_changed>"
evidence:
  - type: eip_state
    content: "<describe-addresses output>"
severity: MEDIUM
mitigation:
  immediate: "Associate EIP or fix routing"
  long_term: "Use EIPs for instances needing static IPs, clean up unused EIPs"
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

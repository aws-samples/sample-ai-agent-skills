---
title: "F4 — Cross-Region Peering Issues"
description: "Diagnose connectivity issues with cross-region VPC peering or TGW peering"
status: active
severity: MEDIUM
triggers:
  - "Cross-region connectivity failure"
  - "Inter-region peering not working"
  - "Cross-region TGW peering"
owner: devops-agent
objective: "Fix cross-region peering connectivity"
context: "Cross-region VPC peering: no SG references, no IPv6 via peering (unless enabled), data transfer charges apply. Cross-region TGW peering: separate TGW route tables, bandwidth limits, inter-region data charges."
---

## Phase 1 — Triage

MUST:
- Check peering connection state and verify it's cross-region
- Verify routes in BOTH regions' route tables point to the peering connection
- Check security groups use CIDR-based rules (not SG references — not supported cross-region)
- Verify NACLs in both regions allow the traffic

SHOULD:
- Check DNS resolution settings on the peering connection
- Verify MTU is 1500 (jumbo frames not supported across peering)
- Check for latency expectations (cross-region adds significant latency)

MAY:
- Measure actual latency and compare to expected inter-region latency
- Check if TGW inter-region peering would be more appropriate

## Common Issues

- symptoms: "Cross-region peering active but SG reference rules don't work"
  diagnosis: "SG-to-SG references are not supported across cross-region peering."
  resolution: "Replace SG references with CIDR-based rules."

- symptoms: "Jumbo frames causing packet drops across regions"
  diagnosis: "MTU is clamped to 1500 across peering connections."
  resolution: "Set MTU to 1500 or enable Path MTU Discovery."

## Safety Ratings

```
safety_ratings:
  - "describe-vpc-peering-connections: GREEN — read-only peering status check"
  - "describe-route-tables: GREEN — read-only route table inspection"
  - "describe-security-groups: GREEN — read-only SG rule inspection"
  - "create-route (add cross-region peering route): YELLOW — adds route, recoverable by deleting"
  - "authorize-security-group-ingress (add CIDR-based rule): YELLOW — adds SG rule, recoverable by revoking"
  - "modify-vpc-peering-connection-options (enable DNS): YELLOW — changes peering option, recoverable by disabling"
```

## Escalation Conditions

- "Fix requires modifying production VPC route tables in multiple regions"
- "Fix requires replacing SG references with CIDR-based rules across multiple SGs"
- "Cross-region latency may impact application performance"
- "Fix involves MTU changes — affects all traffic on affected instances"

## Data Sensitivity

- HIGH: route table entries (expose cross-region network architecture)
- HIGH: security group rules (expose cross-region access patterns)
- MEDIUM: peering connection configurations, MTU settings, latency measurements

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER delete cross-region peering without confirming both regions are aware"

## Phase 3 — Rollback

- If a cross-region route was added: delete it with `aws ec2 delete-route --route-table-id <rtb-id> --destination-cidr-block <peer-cidr>` (in both regions)
- If SG rules were changed from SG references to CIDR: revert by revoking CIDR rule and re-adding SG reference (if reverting to peering)
- If DNS resolution was enabled on peering: disable it with `aws ec2 modify-vpc-peering-connection-options`
- If MTU was changed on instances: revert with `ip link set dev <interface> mtu <original-mtu>`
- Document route tables, SG rules, and MTU settings in BOTH regions before changes

## Output Format

```yaml
root_cause: "cross_region_peering — <detail>"
evidence:
  - type: peering_connection
    content: "<peering state and cross-region configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix SG rules to use CIDRs, adjust MTU"
  long_term: "Document cross-region limitations, use TGW for complex topologies"
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

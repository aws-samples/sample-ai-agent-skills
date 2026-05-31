---
title: "K1 — MTU / Fragmentation Issues"
description: "Diagnose connectivity issues caused by MTU mismatches and packet fragmentation"
status: active
severity: MEDIUM
triggers:
  - "Large packets dropped"
  - "Connection works for small data but fails for large"
  - "Path MTU discovery"
  - "Jumbo frames"
owner: devops-agent
objective: "Fix MTU-related connectivity issues"
context: "Default MTU: 1500 bytes. Jumbo frames (9001) supported within same VPC only. MTU clamped to 1500 through: IGW, NAT gateway, VPN, VPC peering, TGW. If DF bit is set and packet exceeds path MTU, it's dropped."
---

## Phase 1 — Triage

MUST:
- Check instance MTU setting: `ip link show` (Linux) or `netsh interface ipv4 show subinterfaces` (Windows)
- Identify the traffic path (within VPC, across peering, through NAT, through VPN)
- Determine if jumbo frames are configured (MTU 9001)
- Check if the issue only affects large packets/transfers

SHOULD:
- Test with different packet sizes: `ping -M do -s 1472 <dest>` (1472 + 28 header = 1500)
- Check if Path MTU Discovery (PMTUD) is working (ICMP type 3 code 4 must not be blocked)
- Verify security groups and NACLs allow ICMP (needed for PMTUD)

MAY:
- Check if TCP MSS clamping is configured on VPN
- Test with MTU 1500 to confirm jumbo frames are the issue

## Common Issues

- symptoms: "SSH works but SCP/large transfers fail across VPC peering"
  diagnosis: "Jumbo frames (9001) configured but peering clamps to 1500. Large packets dropped."
  resolution: "Set MTU to 1500 on instances that communicate across peering, or enable PMTUD."

- symptoms: "VPN transfers fail for large files"
  diagnosis: "VPN MTU is lower (typically 1400-1436 due to IPsec overhead). Packets too large."
  resolution: "Set MTU to 1400 on instances using VPN, or enable TCP MSS clamping on VPN device."

## Safety Ratings

```
safety_ratings:
  - "describe-instances: GREEN — read-only instance metadata check"
  - "describe-security-groups: GREEN — read-only SG rule inspection (ICMP for PMTUD)"
  - "describe-network-acls: GREEN — read-only NACL rule inspection (ICMP for PMTUD)"
  - "authorize-security-group-ingress (allow ICMP for PMTUD): YELLOW — adds SG rule, recoverable by revoking"
  - "create-network-acl-entry (allow ICMP for PMTUD): YELLOW — adds NACL rule, recoverable by deleting"
```

## Escalation Conditions

- "Fix requires changing MTU on production instances"
- "Fix requires allowing ICMP in security groups and NACLs (for PMTUD)"
- "VPN MTU changes require coordination with on-premises network team"
- "Jumbo frame issues affecting multiple instances across the VPC"

## Data Sensitivity

- HIGH: VPC flow logs (contain source/destination IPs and ports)
- MEDIUM: instance MTU settings, network interface configurations
- MEDIUM: VPN tunnel MTU and MSS clamping settings

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER set MTU above 9001 (maximum supported in VPC)"

## Phase 3 — Rollback

- If MTU was changed on instances: revert with `ip link set dev <interface> mtu <original-mtu>` (Linux) or `netsh interface ipv4 set subinterface <interface> mtu=<original-mtu>` (Windows)
- If ICMP was allowed in SG: revoke it with `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol icmp --port -1 --cidr <cidr>`
- If ICMP was allowed in NACL: delete the rule with `aws ec2 delete-network-acl-entry --network-acl-id <nacl-id> --rule-number <num> --ingress`
- Document original MTU settings and ICMP rules before changes

## Output Format

```yaml
root_cause: "mtu_fragmentation — <detail>"
evidence:
  - type: network
    content: "<MTU settings and path analysis>"
severity: MEDIUM
mitigation:
  immediate: "Reduce MTU to 1500 (or lower for VPN) on affected instances"
  long_term: "Standardize MTU settings, ensure ICMP is allowed for PMTUD"
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

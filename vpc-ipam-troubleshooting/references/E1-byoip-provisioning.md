---
title: "E1 — BYOIP Provisioning"
description: "Diagnose Bring Your Own IP (BYOIP) provisioning issues with IPAM"
status: active
severity: HIGH
triggers: ["BYOIP failed", "bring your own IP", "RIR verification", "BYOIP provisioning error"]
owner: devops-agent
objective: "Resolve BYOIP provisioning and verification issues"
context: "BYOIP allows using your own public IP ranges in AWS. Requires RIR verification and IPAM pool provisioning."
---

## Phase 1 — Triage

MUST:
- Check BYOIP CIDR status: `aws ec2 describe-byoip-cidrs --max-results 10 --query 'ByoipCidrs[*].{Cidr:Cidr,State:State,StatusMessage:StatusMessage}'`
- Verify the CIDR is provisioned to an IPAM public scope pool
- Check RIR authorization status (ROA — Route Origin Authorization)

SHOULD:
- Verify the X.509 certificate and signed message for CIDR ownership
- Check if the CIDR range meets minimum size requirements (/24 for IPv4, /48 for IPv6)

MAY:
- Review CloudTrail for ProvisionByoipCidr events
- Check if the CIDR is already advertised by another AWS account

## Phase 2 — Remediate

MUST:
- Complete RIR verification (create ROA with Amazon ASN 16509 or 14618)
- Generate and upload X.509 certificate to RIR RDAP record
- Provision BYOIP CIDR with correct signed message and certificate

SHOULD:
- Wait for RIR propagation (can take up to 24 hours)
- Verify ROA is visible in RPKI validators

MAY:
- Use IPAM to manage BYOIP CIDRs alongside Amazon-provided CIDRs
- Configure BYOIP for specific regions

## Common Issues

- symptoms: "ProvisionByoipCidr returns 'verification pending' for extended period"
  diagnosis: "RIR ROA has not propagated or X.509 certificate is not in RDAP record."
  resolution: "Verify ROA is created with correct ASN (16509). Check RDAP record contains the X.509 certificate. Allow 24 hours."

- symptoms: "BYOIP CIDR shows failed-provision state"
  diagnosis: "Signed message is invalid or certificate does not match the CIDR ownership."
  resolution: "Regenerate the signed message with the correct certificate. Verify CIDR ownership in RIR records."

## Safety Ratings

```
safety_ratings:
  - "describe-byoip-cidrs: GREEN — read-only BYOIP status"
  - "provision-byoip-cidr: YELLOW — provisions public IP range"
  - "advertise-byoip-cidr: RED — starts BGP advertisement of your IP range"
  - "deprovision-byoip-cidr: RED — removes BYOIP range from AWS"
```

## Escalation Conditions

- RIR verification fails and requires RIR support contact
- BYOIP CIDR is already provisioned in another AWS account
- ROA creation requires coordination with IP address holder (different team/org)
- BYOIP provisioning needed urgently but RIR propagation is slow

## Data Sensitivity

- `aws ec2 describe-byoip-cidrs`: MEDIUM — reveals owned public IP ranges
- RIR records: MEDIUM — contain organization identity and IP ownership
- X.509 certificates: HIGH — contain cryptographic material for IP verification
- Signed messages: HIGH — contain authorization tokens

## Prohibited Actions

- NEVER share X.509 private keys used for BYOIP verification
- NEVER advertise BYOIP CIDRs before verifying routing is correct
- NEVER deprovision BYOIP CIDRs that are actively in use (Elastic IPs, etc.)
- NEVER provision BYOIP CIDRs without valid RIR authorization

## Phase 3 — Rollback

1. If BYOIP was advertised incorrectly, withdraw advertisement: `aws ec2 withdraw-byoip-cidr --cidr <cidr>`
2. If BYOIP must be deprovisioned: `aws ec2 deprovision-byoip-cidr --cidr <cidr>` (only after withdrawing and releasing all EIPs)
3. If ROA was created incorrectly, update it at the RIR
4. Verify no Elastic IPs or resources depend on the BYOIP range
5. Document the BYOIP provisioning attempt and outcome

## Output Format

```yaml
root_cause: "byoip_provisioning — <specific_cause>"
evidence:
  - type: byoip_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix RIR verification or certificate"
  long_term: "Document BYOIP provisioning process"
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

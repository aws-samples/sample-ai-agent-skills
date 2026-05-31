---
title: "E2 — Public IP Insights"
description: "Diagnose IPAM Public IP Insights issues"
status: active
severity: MEDIUM
triggers: ["public IP insights", "public IP usage", "EIP tracking", "public IP not showing"]
owner: devops-agent
objective: "Resolve Public IP Insights visibility and reporting issues"
context: "Public IP Insights provides visibility into public IP usage across the organization. Requires IPAM Advanced tier."
---

## Phase 1 — Triage

MUST:
- Verify IPAM exists and is configured: `aws ec2 describe-ipams --query 'Ipams[*].{IpamId:IpamId,State:State,Tier:Tier}'`
- Check IPAM tier (Advanced tier required for Public IP Insights)
- Verify resource discovery is configured for target regions

SHOULD:
- Check discovered public IPs: `aws ec2 get-ipam-discovered-public-addresses --ipam-resource-discovery-id <discovery-id> --address-region <region>`
- Verify operating regions include all regions with public IPs

MAY:
- Review public IP usage across accounts
- Check for unassociated Elastic IPs

## Phase 2 — Remediate

MUST:
- Upgrade IPAM to Advanced tier if on Free tier
- Configure resource discovery for all regions with public IP resources
- Ensure IPAM operating regions cover all target regions

SHOULD:
- Set up regular public IP audits using Insights data
- Identify and release unassociated Elastic IPs

MAY:
- Create dashboards for public IP usage tracking
- Implement alerts for unexpected public IP allocation

## Common Issues

- symptoms: "Public IP Insights shows no data"
  diagnosis: "IPAM is on Free tier or resource discovery is not configured."
  resolution: "Upgrade to Advanced tier: aws ec2 modify-ipam --ipam-id <id> --tier advanced. Configure resource discovery."

- symptoms: "Public IPs from some regions are missing"
  diagnosis: "Those regions are not in IPAM operating regions or resource discovery."
  resolution: "Add missing regions to IPAM operating regions and resource discovery."

## Safety Ratings

```
safety_ratings:
  - "describe-ipams: GREEN — read-only IPAM details"
  - "get-ipam-discovered-public-addresses: GREEN — read-only public IP listing"
  - "modify-ipam tier: YELLOW — changes IPAM tier (affects pricing)"
  - "release-address (EIP): RED — releases public IP address"
```

## Escalation Conditions

- Advanced tier costs are prohibitive for the organization
- Public IP Insights shows unexpected public IPs requiring security investigation
- Resource discovery fails to detect public IPs in specific accounts
- Compliance requires complete public IP inventory but coverage is incomplete

## Data Sensitivity

- `aws ec2 describe-ipams`: LOW — returns IPAM metadata
- `aws ec2 get-ipam-discovered-public-addresses`: HIGH — reveals all public IP addresses and their associations
- Public IP inventory: HIGH — reveals external attack surface
- EIP-to-resource mapping: MEDIUM — reveals infrastructure topology

## Prohibited Actions

- NEVER release Elastic IPs without verifying they are not in use
- NEVER share public IP inventory data outside the security team
- NEVER downgrade IPAM tier without understanding the impact on monitoring
- NEVER ignore unexpected public IPs — they may indicate security issues

## Phase 3 — Rollback

1. If IPAM was upgraded to Advanced tier and cost is a concern, downgrade: `aws ec2 modify-ipam --ipam-id <id> --tier free` (loses Advanced features)
2. If resource discovery was modified, revert operating regions
3. Verify Public IP Insights data is consistent after changes
4. Document the public IP inventory and any anomalies found
5. Update security team on public IP posture changes

## Output Format

```yaml
root_cause: "public_ip_insights — <specific_cause>"
evidence:
  - type: insights_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix IPAM tier or resource discovery"
  long_term: "Implement continuous public IP monitoring"
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

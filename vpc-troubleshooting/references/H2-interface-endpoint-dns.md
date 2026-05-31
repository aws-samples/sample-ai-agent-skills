---
title: "H2 — Interface Endpoint DNS Issues"
description: "Diagnose interface VPC endpoint DNS resolution and connectivity failures"
status: active
severity: HIGH
triggers:
  - "Interface endpoint not resolving"
  - "AWS service call failing from VPC"
  - "Private DNS not working for endpoint"
owner: devops-agent
objective: "Fix interface endpoint DNS resolution and connectivity"
context: "Interface endpoints create ENIs in subnets. Private DNS (enabled by default) overrides the public service DNS to resolve to the endpoint ENI IPs. Requires enableDnsSupport and enableDnsHostnames on the VPC."
---

## Phase 1 — Triage

MUST:
- Check interface endpoint state: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <vpce-id>`
- Verify private DNS is enabled on the endpoint (PrivateDnsEnabled=true)
- Check VPC has enableDnsSupport=true and enableDnsHostnames=true
- Verify endpoint is in subnets that can reach the target resources

SHOULD:
- Test DNS resolution of the service endpoint from within the VPC
- Check security group on the endpoint ENI allows inbound on the service port (usually 443)
- Verify the endpoint policy allows the required actions

MAY:
- Check if Route 53 Resolver rules override the endpoint DNS
- Test using the endpoint-specific DNS name directly (vpce-xxx.service.region.vpce.amazonaws.com)

## Common Issues

- symptoms: "Private DNS enabled but service still resolves to public IP"
  diagnosis: "enableDnsHostnames not enabled on VPC, or DNS cache stale."
  resolution: "Enable enableDnsHostnames on VPC, flush DNS cache."

- symptoms: "Endpoint exists but connection times out"
  diagnosis: "Security group on endpoint ENI doesn't allow inbound HTTPS (443)."
  resolution: "Add inbound rule for TCP 443 from the VPC CIDR to the endpoint security group."

## Safety Ratings

```
safety_ratings:
  - "describe-vpc-endpoints: GREEN — read-only endpoint state inspection"
  - "describe-vpc-attribute: GREEN — read-only VPC DNS attribute check"
  - "describe-security-groups: GREEN — read-only SG rule inspection"
  - "modify-vpc-attribute (enable DNS hostnames): YELLOW — changes VPC setting, recoverable by disabling"
  - "authorize-security-group-ingress (allow 443 on endpoint SG): YELLOW — adds SG rule, recoverable by revoking"
  - "modify-vpc-endpoint (enable private DNS): YELLOW — changes DNS setting, recoverable by disabling"
```

## Escalation Conditions

- "Fix requires modifying VPC DNS attributes in production"
- "Fix requires changing security groups on interface endpoint ENIs"
- "Private DNS change affects all service resolution in the VPC"
- "Multiple services affected by endpoint DNS resolution failure"

## Data Sensitivity

- HIGH: endpoint ENI IPs and configurations (expose private endpoint architecture)
- HIGH: security group rules on endpoint (expose access patterns)
- MEDIUM: VPC DNS settings, endpoint policy, private DNS configuration

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER disable private DNS on an endpoint without confirming applications use endpoint-specific DNS"

## Phase 3 — Rollback

- If enableDnsHostnames was enabled: disable it with `aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --no-enable-dns-hostnames`
- If a security group rule was added to endpoint: revoke it with `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol tcp --port 443 --cidr <cidr>`
- If private DNS was enabled on endpoint: disable it with `aws ec2 modify-vpc-endpoint --vpc-endpoint-id <vpce-id> --private-dns-enabled false` (WARNING: may break service access)
- Document VPC DNS attributes, endpoint SG rules, and DNS settings before changes

## Output Format

```yaml
root_cause: "interface_endpoint_dns — <detail>"
evidence:
  - type: vpc_endpoint
    content: "<endpoint state, DNS settings, security group>"
severity: HIGH
mitigation:
  immediate: "Fix DNS settings or security group on endpoint"
  long_term: "Standardize endpoint deployment with IaC, monitor endpoint health"
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

---
title: "Z1 — General Route 53 Resolver Troubleshooting"
description: "Catch-all runbook for Resolver issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["Route 53 Resolver error", "DNS resolver issue", "general resolver problem"]
owner: devops-agent
objective: "Provide systematic troubleshooting for unclassified Resolver issues"
context: "This runbook covers general Route 53 Resolver issues. Start here when the issue is unclear, then route to a specific runbook once the domain is identified."
---

## Phase 1 — Triage

MUST:
- List all resolver endpoints: `aws route53resolver list-resolver-endpoints --query 'ResolverEndpoints[*].{Id:Id,Name:Name,Direction:Direction,Status:Status}'`
- List all forwarding rules: `aws route53resolver list-resolver-rules --query 'ResolverRules[*].{Id:Id,DomainName:DomainName,Status:Status}'`
- List DNS Firewall rule groups: `aws route53resolver list-firewall-rule-groups`

SHOULD:
- Check CloudTrail for recent Resolver API errors
- Verify VPC DNS settings: `aws ec2 describe-vpcs --vpc-ids <vpc-id> --query 'Vpcs[*].{DnsSupport:EnableDnsSupport,DnsHostnames:EnableDnsHostnames}'`

MAY:
- Check AWS Health Dashboard for Route 53 Resolver service issues
- Review recent infrastructure changes affecting DNS

## Phase 2 — Remediate

MUST:
- Identify the specific issue domain (endpoints, forwarding, firewall, logging, DNSSEC)
- Route to the appropriate specific runbook (A1-G1) once domain is identified
- Verify basic prerequisites: VPC DNS enabled, endpoints operational, rules associated

SHOULD:
- Check if the issue affects a single VPC or multiple VPCs
- Determine if the issue started after a recent change

MAY:
- Enable query logging for detailed DNS analysis
- Open an AWS Support case if the issue appears service-side

## Common Issues

- symptoms: "DNS resolution not working in VPC"
  diagnosis: "VPC DNS resolution disabled or DHCP option set misconfigured."
  resolution: "Enable DNS resolution on VPC. Verify DHCP option set has correct DNS settings."

- symptoms: "Some DNS queries work but others fail"
  diagnosis: "Forwarding rules or DNS Firewall blocking specific domains."
  resolution: "Check forwarding rules and DNS Firewall rules for the failing domains."

## Safety Ratings

```
safety_ratings:
  - "list-resolver-endpoints: GREEN — read-only listing"
  - "list-resolver-rules: GREEN — read-only listing"
  - "list-firewall-rule-groups: GREEN — read-only listing"
  - "describe-vpcs: GREEN — read-only VPC information"
```

## Escalation Conditions

- Complete DNS resolution failure in a VPC
- Multiple unrelated Resolver features fail simultaneously
- Issue affects multiple VPCs or accounts
- Service health shows active incidents

## Data Sensitivity

- Resolver endpoint listings: LOW — endpoint metadata only
- Forwarding rules: MEDIUM — reveal DNS architecture and target servers
- Firewall rule groups: MEDIUM — reveal security policy
- VPC DNS settings: LOW — basic VPC configuration

## Prohibited Actions

- NEVER make broad DNS changes without isolating the issue first
- NEVER disable DNS Firewall to troubleshoot resolution issues
- NEVER assume a service outage without checking AWS Health Dashboard
- NEVER skip triage and jump directly to remediation

## Phase 3 — Rollback

1. Document all changes made during troubleshooting
2. Revert changes in reverse order if they did not resolve the issue
3. Verify DNS resolution is in the same state as before troubleshooting
4. Collect all diagnostic data for escalation if unresolved
5. Document troubleshooting steps and results

## Output Format

```yaml
root_cause: "general_resolver — <specific_cause>"
evidence:
  - type: general_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook or apply general fix"
  long_term: "Implement DNS monitoring and alerting"
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
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling DNSSEC without understanding impact"
  - "NEVER suggest wildcard DNS records without review"
  - "NEVER modify hosted zones without backup"

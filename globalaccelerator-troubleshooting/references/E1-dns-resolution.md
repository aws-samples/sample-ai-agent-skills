---
title: "E1 — Global Accelerator DNS Resolution Issues"
description: "Diagnose DNS resolution problems for Global Accelerator"
status: active
severity: HIGH
triggers:
  - "DNS resolution failure"
  - "cannot resolve accelerator"
  - "DNS not working"
  - "accelerator DNS name"
owner: devops-agent
objective: "Resolve DNS resolution issues for Global Accelerator accelerators"
context: "Each accelerator gets a DNS name (e.g., a1234567890abcdef0.awsglobalaccelerator.com) that resolves to the 2 static anycast IPs. Custom domain names require CNAME or alias records pointing to the accelerator DNS name."
---

## Phase 1 — Triage

MUST:
- Get accelerator DNS name: `aws globalaccelerator describe-accelerator --accelerator-arn <accelerator-arn> --query 'Accelerator.DnsName'`
- Test DNS resolution: `nslookup <accelerator-dns-name>` and `dig <accelerator-dns-name>`
- Verify accelerator is enabled and deployed: `aws globalaccelerator describe-accelerator --accelerator-arn <accelerator-arn> --query '{Status:Accelerator.Status,Enabled:Accelerator.Enabled}'`
- If using custom domain, check DNS records: `dig <custom-domain> CNAME` and `dig <custom-domain> A`

SHOULD:
- Verify the accelerator DNS name resolves to the 2 static anycast IPs
- Check if custom domain CNAME points to the correct accelerator DNS name
- Test DNS resolution from multiple locations/DNS servers

MAY:
- Check Route 53 hosted zone if using Route 53 for DNS: `aws route53 list-resource-record-sets --hosted-zone-id <zone-id> --query "ResourceRecordSets[?Name=='<custom-domain>.']"`
- Verify DNS propagation if records were recently changed

## Phase 2 — Remediate

MUST:
- Ensure accelerator is enabled and in DEPLOYED status
- Create correct CNAME or alias record for custom domains pointing to accelerator DNS name
- Verify DNS TTL allows for timely propagation

SHOULD:
- Use Route 53 alias records instead of CNAME for zone apex domains
- Set appropriate DNS TTL (lower for initial setup, higher for production)

MAY:
- Configure health checks on the DNS records for additional failover

## Common Issues

- symptoms: "Custom domain does not resolve to accelerator IPs"
  diagnosis: "CNAME record is missing or points to wrong accelerator DNS name."
  resolution: "Create CNAME record pointing to the accelerator DNS name."

- symptoms: "DNS resolves but connection fails"
  diagnosis: "Accelerator is disabled or no listeners configured."
  resolution: "Enable the accelerator and configure listeners."

- symptoms: "DNS resolution returns old IPs after accelerator recreation"
  diagnosis: "DNS cache still has old records due to high TTL."
  resolution: "Wait for DNS TTL to expire or flush DNS cache."

## Output Format

```yaml
root_cause: "dns_resolution — <specific_cause>"
evidence:
  - type: accelerator_dns
    content: "<accelerator DNS name>"
  - type: dns_resolution
    content: "<nslookup/dig results>"
  - type: custom_domain
    content: "<CNAME/alias record configuration>"
severity: HIGH
mitigation:
  immediate: "Fix DNS records or enable accelerator"
  long_term: "Use Route 53 alias records and monitor DNS health"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never disable or delete an accelerator serving production traffic without confirming failover
- Never remove endpoint groups without verifying traffic has been drained
- Never modify static IP associations on active accelerators
- Never change routing configurations during active incidents without approval
- Never delete flow logs that may be needed for audit or investigation

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Re-enable previous endpoint group configurations if traffic routing degrades
- Restore traffic dial settings to previous values if performance worsens
- Revert client affinity settings if session persistence breaks

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"

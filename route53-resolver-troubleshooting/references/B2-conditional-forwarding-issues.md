---
title: "B2 — Conditional Forwarding Issues"
description: "Diagnose conditional DNS forwarding configuration and resolution issues"
status: active
severity: HIGH
triggers: ["conditional forwarding broken", "domain not forwarding", "split DNS not working"]
owner: devops-agent
objective: "Resolve conditional DNS forwarding issues for hybrid DNS architectures"
context: "Conditional forwarding uses resolver rules to forward queries for specific domains to designated DNS servers. This enables split-horizon DNS where some domains resolve via on-premises DNS and others via AWS."
---

## Phase 1 — Triage

MUST:
- List all forwarding rules and their domains: `aws route53resolver list-resolver-rules --query 'ResolverRules[*].{Id:Id,DomainName:DomainName,RuleType:RuleType,Status:Status}'`
- Check rule associations: `aws route53resolver list-resolver-rule-associations --filters Name=VPCId,Values=<vpc-id>`
- Verify outbound endpoint status: `aws route53resolver get-resolver-endpoint --resolver-endpoint-id <endpoint-id>`

SHOULD:
- Test DNS resolution for the forwarded domain: `dig @<vpc-dns> <domain>`
- Verify target DNS servers respond to queries for the domain

MAY:
- Enable query logging to trace query resolution path
- Check if multiple rules match the same domain (most specific wins)

## Phase 2 — Remediate

MUST:
- Ensure the forwarding rule domain matches exactly (e.g., `corp.example.com.` not `example.com.`)
- Verify the rule is associated with the correct VPC
- Confirm target DNS servers are reachable from the outbound endpoint

SHOULD:
- Create more specific rules for subdomains that need different forwarding
- Verify no system rules conflict with the forwarding configuration

MAY:
- Implement DNS query logging to monitor forwarding behavior
- Set up health checks for target DNS servers

## Common Issues

- symptoms: "Queries for subdomain.corp.example.com not forwarding but corp.example.com works"
  diagnosis: "A more specific rule or system rule is matching the subdomain."
  resolution: "Check for conflicting rules. Create a specific rule for the subdomain if needed."

- symptoms: "Conditional forwarding works for some instances but not others in the same VPC"
  diagnosis: "Instances using custom DNS settings instead of VPC DNS resolver."
  resolution: "Ensure instances use the VPC DNS resolver (DHCP option set) not custom /etc/resolv.conf."

## Safety Ratings

```
safety_ratings:
  - "list-resolver-rules: GREEN — read-only rule listing"
  - "list-resolver-rule-associations: GREEN — read-only association listing"
  - "create-resolver-rule: YELLOW — changes DNS resolution behavior"
  - "update-resolver-rule: YELLOW — modifies existing DNS forwarding path"
```

## Escalation Conditions

- Conditional forwarding causes intermittent DNS failures for critical applications
- Split-horizon DNS configuration conflicts between AWS and on-premises
- Multiple overlapping rules create unpredictable forwarding behavior
- Target DNS servers are overloaded by forwarded queries

## Data Sensitivity

- `aws route53resolver list-resolver-rules`: LOW — rule metadata and domain names
- DNS query results: MEDIUM — may reveal internal hostnames and IP addresses
- Target DNS server IPs: MEDIUM — reveals on-premises DNS infrastructure
- Query logs: HIGH — contain all DNS queries including internal service names

## Prohibited Actions

- NEVER create a catch-all forwarding rule for "." (root domain) without understanding the impact
- NEVER modify forwarding rules during business hours without a maintenance window
- NEVER assume DNS propagation is instant — allow time for cache expiration
- NEVER delete conditional forwarding rules without verifying no applications depend on them

## Phase 3 — Rollback

1. If a rule was modified incorrectly, update it back: `aws route53resolver update-resolver-rule --resolver-rule-id <rule-id> --config <original-config>`
2. If a new rule was created that conflicts, delete it
3. If VPC associations were changed, restore original associations
4. Flush DNS caches on affected instances if needed
5. Document the conditional forwarding architecture for future reference

## Output Format

```yaml
root_cause: "conditional_forwarding — <specific_cause>"
evidence:
  - type: forwarding_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix conditional forwarding configuration"
  long_term: "Document and monitor split-horizon DNS architecture"
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

---
title: "B1 — Forwarding Rule Errors"
description: "Diagnose resolver forwarding rule creation and operation errors"
status: active
severity: HIGH
triggers: ["forwarding rule failed", "rule not working", "DNS not forwarding"]
owner: devops-agent
objective: "Resolve forwarding rule configuration and operation issues"
context: "Forwarding rules direct DNS queries for specific domains to target DNS servers via outbound endpoints. Rules must be associated with VPCs to take effect. System rules for AWS domains cannot be overridden."
---

## Phase 1 — Triage

MUST:
- List forwarding rules: `aws route53resolver list-resolver-rules --query 'ResolverRules[*].{Id:Id,Name:Name,DomainName:DomainName,Status:Status,RuleType:RuleType}'`
- Check rule details: `aws route53resolver get-resolver-rule --resolver-rule-id <rule-id>`
- Verify rule is associated with the VPC: `aws route53resolver list-resolver-rule-associations --query 'ResolverRuleAssociations[*].{RuleId:ResolverRuleId,VPCId:VPCId,Status:Status}'`

SHOULD:
- Verify the outbound endpoint referenced by the rule is operational
- Check target DNS server IP addresses are correct and reachable

MAY:
- Test DNS resolution for the forwarded domain from an instance in the VPC
- Review query logs to see if queries are reaching the forwarding rule

## Phase 2 — Remediate

MUST:
- Associate rule with VPC if not associated: `aws route53resolver associate-resolver-rule --resolver-rule-id <rule-id> --vpc-id <vpc-id>`
- Verify outbound endpoint is in OPERATIONAL status
- Ensure target DNS server IPs are correct and responsive

SHOULD:
- Check that the domain name in the rule matches the queries being made (including trailing dot)
- Verify the outbound endpoint security group allows outbound DNS to target servers

MAY:
- Create more specific rules for subdomains if needed
- Enable query logging to verify rule matching

## Common Issues

- symptoms: "Forwarding rule created but DNS queries not being forwarded"
  diagnosis: "Rule not associated with the VPC where queries originate."
  resolution: "Associate the rule with the correct VPC."

- symptoms: "Forwarding rule associated but queries return SERVFAIL"
  diagnosis: "Target DNS servers unreachable or not responding."
  resolution: "Verify target DNS server IPs. Check outbound endpoint connectivity to target servers."

## Safety Ratings

```
safety_ratings:
  - "list-resolver-rules: GREEN — read-only rule listing"
  - "get-resolver-rule: GREEN — read-only rule details"
  - "associate-resolver-rule: YELLOW — changes DNS resolution behavior for the VPC"
  - "create-resolver-rule: YELLOW — creates new DNS forwarding path"
```

## Escalation Conditions

- Forwarding rule causes DNS resolution failures for critical services
- System rules conflict with required forwarding configuration
- Target DNS servers are managed by a different team and are unresponsive
- Rule association fails with permission or limit errors

## Data Sensitivity

- `aws route53resolver list-resolver-rules`: LOW — rule metadata and domain names
- `aws route53resolver get-resolver-rule`: MEDIUM — reveals target DNS server IPs
- Rule associations: LOW — VPC and rule ID mappings
- Target DNS server IPs: MEDIUM — reveals on-premises DNS infrastructure

## Prohibited Actions

- NEVER create forwarding rules for amazonaws.com or other AWS internal domains
- NEVER delete forwarding rules in production without verifying impact on DNS resolution
- NEVER point forwarding rules to untrusted or unverified DNS servers
- NEVER create overly broad rules (e.g., for ".") that forward all DNS queries

## Phase 3 — Rollback

1. If a rule was associated incorrectly, disassociate it: `aws route53resolver disassociate-resolver-rule --resolver-rule-id <rule-id> --vpc-id <vpc-id>`
2. If a rule was created incorrectly, delete it: `aws route53resolver delete-resolver-rule --resolver-rule-id <rule-id>`
3. If target IPs were changed, update them back to the original values
4. Verify DNS resolution returns to normal after rollback
5. Document the changes and notify affected teams

## Output Format

```yaml
root_cause: "forwarding_rule — <specific_cause>"
evidence:
  - type: rule_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix forwarding rule configuration"
  long_term: "Implement DNS monitoring and rule validation"
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

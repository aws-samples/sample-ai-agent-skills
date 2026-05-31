---
title: "C1 — DNS Firewall Rule Group Issues"
description: "Diagnose DNS Firewall rule group creation, configuration, and evaluation issues"
status: active
severity: HIGH
triggers: ["DNS Firewall not blocking", "firewall rule group error", "DNS Firewall configuration"]
owner: devops-agent
objective: "Resolve DNS Firewall rule group configuration and operation issues"
context: "DNS Firewall rule groups contain rules that ALLOW, BLOCK, or ALERT on DNS queries matching domain lists. Rule groups must be associated with VPCs. Rules are evaluated by priority (lowest number first)."
---

## Phase 1 — Triage

MUST:
- List firewall rule groups: `aws route53resolver list-firewall-rule-groups --query 'FirewallRuleGroups[*].{Id:Id,Name:Name,ShareStatus:ShareStatus}'`
- Get rule group details: `aws route53resolver get-firewall-rule-group --firewall-rule-group-id <group-id>`
- List rules in the group: `aws route53resolver list-firewall-rules --firewall-rule-group-id <group-id> --query 'FirewallRules[*].{Priority:Priority,Action:Action,DomainListId:FirewallDomainListId}'`

SHOULD:
- Check VPC associations: `aws route53resolver list-firewall-rule-group-associations --query 'FirewallRuleGroupAssociations[*].{GroupId:FirewallRuleGroupId,VPCId:VpcId,Priority:Priority,Status:Status}'`
- Verify domain lists contain the expected domains

MAY:
- Enable DNS Firewall logs to see which rules are matching
- Check the fail-open/fail-close configuration for the VPC

## Phase 2 — Remediate

MUST:
- Associate rule group with VPC if not associated: `aws route53resolver associate-firewall-rule-group --creator-request-id <id> --firewall-rule-group-id <group-id> --vpc-id <vpc-id> --priority <priority>`
- Verify rule priorities are ordered correctly (lowest number = highest priority)
- Ensure domain lists are populated with the correct domains

SHOULD:
- Review rule evaluation order to prevent unintended blocks or allows
- Configure DNS Firewall logging for visibility

MAY:
- Use managed domain lists (AWS threat intelligence) for common threats
- Implement fail-close for high-security VPCs

## Common Issues

- symptoms: "DNS Firewall rule group created but not blocking any queries"
  diagnosis: "Rule group not associated with the VPC."
  resolution: "Associate the rule group with the target VPC."

- symptoms: "DNS Firewall blocking legitimate domains"
  diagnosis: "Overly broad domain list or incorrect rule priority ordering."
  resolution: "Add ALLOW rules with higher priority (lower number) for legitimate domains."

## Safety Ratings

```
safety_ratings:
  - "list-firewall-rule-groups: GREEN — read-only listing"
  - "list-firewall-rules: GREEN — read-only rule listing"
  - "associate-firewall-rule-group: YELLOW — changes DNS filtering for the VPC"
  - "create-firewall-rule BLOCK: RED — blocks DNS resolution for matched domains"
```

## Escalation Conditions

- DNS Firewall blocks critical application DNS queries causing outage
- Rule priority conflicts between multiple rule groups on the same VPC
- Managed domain lists block legitimate third-party services
- Fail-close configuration causes complete DNS outage during firewall issues

## Data Sensitivity

- `aws route53resolver list-firewall-rule-groups`: LOW — group metadata only
- `aws route53resolver list-firewall-rules`: MEDIUM — reveals security policy (blocked domains)
- Domain lists: MEDIUM — reveal security threat intelligence and policy
- DNS Firewall logs: HIGH — contain all DNS queries and firewall actions

## Prohibited Actions

- NEVER disable DNS Firewall entirely to troubleshoot DNS resolution issues
- NEVER set fail-close without understanding the impact on DNS availability
- NEVER create BLOCK rules for broad TLDs (.com, .org) without specific justification
- NEVER modify shared rule groups without coordinating with the sharing account

## Phase 3 — Rollback

1. If a rule group was associated incorrectly, disassociate it: `aws route53resolver disassociate-firewall-rule-group --firewall-rule-group-association-id <id>`
2. If rules were added that block legitimate traffic, delete them: `aws route53resolver delete-firewall-rule --firewall-rule-group-id <group-id> --firewall-domain-list-id <list-id>`
3. If fail-close was enabled, revert to fail-open: `aws route53resolver update-firewall-config --resource-id <vpc-id> --firewall-fail-open ENABLED`
4. Verify DNS resolution is restored for affected domains
5. Document the firewall changes and notify the security team

## Output Format

```yaml
root_cause: "dns_firewall_rule_group — <specific_cause>"
evidence:
  - type: firewall_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix DNS Firewall configuration"
  long_term: "Implement firewall rule testing and change management"
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

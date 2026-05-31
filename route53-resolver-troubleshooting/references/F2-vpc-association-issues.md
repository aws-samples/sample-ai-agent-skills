---
title: "F2 — VPC Association Issues"
description: "Diagnose VPC association issues for resolver rules and DNS Firewall rule groups"
status: active
severity: HIGH
triggers: ["VPC association failed", "rule not applied to VPC", "firewall not active in VPC"]
owner: devops-agent
objective: "Resolve VPC association issues for resolver rules and DNS Firewall rule groups"
context: "Resolver rules and DNS Firewall rule groups must be associated with VPCs to take effect. Association failures prevent DNS forwarding or firewall filtering from working in the target VPC."
---

## Phase 1 — Triage

MUST:
- Check rule associations: `aws route53resolver list-resolver-rule-associations --query 'ResolverRuleAssociations[*].{RuleId:ResolverRuleId,VPCId:VPCId,Status:Status,StatusMessage:StatusMessage}'`
- Check firewall associations: `aws route53resolver list-firewall-rule-group-associations --query 'FirewallRuleGroupAssociations[*].{GroupId:FirewallRuleGroupId,VPCId:VpcId,Status:Status,Priority:Priority}'`
- Verify VPC exists and has DNS resolution enabled: `aws ec2 describe-vpcs --vpc-ids <vpc-id> --query 'Vpcs[*].{Id:VpcId,DnsResolution:EnableDnsSupport,DnsHostnames:EnableDnsHostnames}'`

SHOULD:
- Check if the VPC already has the maximum number of associations
- Verify IAM permissions for association operations

MAY:
- Review CloudTrail for association errors
- Check if the resource is shared via RAM and the share is active

## Phase 2 — Remediate

MUST:
- Associate rule with VPC: `aws route53resolver associate-resolver-rule --resolver-rule-id <rule-id> --vpc-id <vpc-id> --name <name>`
- For firewall: `aws route53resolver associate-firewall-rule-group --creator-request-id <id> --firewall-rule-group-id <group-id> --vpc-id <vpc-id> --priority <priority>`
- Ensure DNS resolution is enabled on the VPC

SHOULD:
- Verify association status transitions to COMPLETE
- Test DNS resolution after association

MAY:
- Automate VPC associations via CloudFormation or Terraform
- Set up notifications for association status changes

## Common Issues

- symptoms: "Association fails with 'already associated' error"
  diagnosis: "The rule or firewall group is already associated with the VPC."
  resolution: "Check existing associations. Update or disassociate the existing one if needed."

- symptoms: "Association succeeds but DNS behavior unchanged"
  diagnosis: "Rule domain doesn't match queries or firewall rule priorities are incorrect."
  resolution: "Verify rule domain matches. Check firewall rule priorities and domain lists."

## Safety Ratings

```
safety_ratings:
  - "list-resolver-rule-associations: GREEN — read-only listing"
  - "list-firewall-rule-group-associations: GREEN — read-only listing"
  - "associate-resolver-rule: YELLOW — changes DNS resolution for the VPC"
  - "associate-firewall-rule-group: YELLOW — enables DNS filtering for the VPC"
```

## Escalation Conditions

- Association fails with service errors despite correct configuration
- VPC association limit reached
- Shared resources cannot be associated in the target account
- Association changes cause DNS resolution failures in the VPC

## Data Sensitivity

- Association listings: LOW — VPC and rule/group ID mappings
- VPC details: MEDIUM — reveal network architecture
- Association operations: MEDIUM — modify DNS behavior for the VPC
- Firewall priorities: MEDIUM — reveal security policy ordering

## Prohibited Actions

- NEVER disassociate rules or firewall groups from production VPCs without testing
- NEVER associate untested firewall rule groups with production VPCs
- NEVER ignore association status errors — they indicate configuration problems
- NEVER create circular or conflicting associations across multiple VPCs

## Phase 3 — Rollback

1. If a rule was associated incorrectly, disassociate: `aws route53resolver disassociate-resolver-rule --resolver-rule-id <rule-id> --vpc-id <vpc-id>`
2. If a firewall group was associated incorrectly: `aws route53resolver disassociate-firewall-rule-group --firewall-rule-group-association-id <id>`
3. Verify DNS resolution returns to normal after disassociation
4. Check that no dependent services are affected
5. Document association changes

## Output Format

```yaml
root_cause: "vpc_association — <specific_cause>"
evidence:
  - type: association_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix VPC association"
  long_term: "Automate VPC associations via IaC"
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

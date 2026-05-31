---
title: "C2 — Security Group Misconfiguration"
description: "Diagnose connectivity issues caused by security group rules"
status: active
severity: MEDIUM
triggers:
  - "security group.*deny"
  - "traffic.*blocked"
  - "cannot connect.*port"
  - "VPC flow log.*REJECT"
owner: devops-agent
objective: "Identify the security group rule gap and restore connectivity"
context: "Security groups are stateful firewalls at the ENI level. They have ALLOW rules only (no deny). Default: deny all inbound, allow all outbound. Return traffic for allowed connections is automatically permitted."
---

## Phase 1 — Triage

MUST:
- List security groups on the instance: `aws ec2 describe-instances --instance-ids <id>` → SecurityGroups
- Review inbound rules: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Verify the required port/protocol is allowed from the source IP/CIDR/SG
- Check ALL security groups attached to the ENI (rules are aggregated as union)

SHOULD:
- Check VPC flow logs for REJECT entries matching the traffic pattern
- Verify source and destination security groups if using SG-to-SG references
- Check if the SG references another SG in a peered VPC (requires peering SG reference)

MAY:
- Use VPC Reachability Analyzer to validate the path

## Guardrails

- Security groups have NO deny rules. You cannot block specific IPs with SGs. Use NACLs for deny.
- Security groups are STATEFUL. If inbound is allowed, return traffic is auto-allowed. Don't add outbound rules for return traffic.
- SG rules referencing another SG only work within the same VPC (or peered VPC with SG reference enabled).
- Maximum 5 SGs per ENI (adjustable). Rules from all SGs are combined.

## Common Issues

- symptoms: "Inbound traffic blocked, SG appears to have correct rules"
  diagnosis: "Source IP may not match the rule. Check if source is behind NAT (public IP differs from private IP)."
  resolution: "Add rule for the actual source IP/CIDR. For internet traffic, use the public IP."

- symptoms: "Outbound traffic blocked"
  diagnosis: "Default SG allows all outbound. If custom SG removed the default outbound rule, specific outbound rules are needed."
  resolution: "Add outbound rule for the required destination and port."

- symptoms: "SG-to-SG reference not working across peered VPCs"
  diagnosis: "Cross-VPC SG references require VPC peering with 'allow remote SG references' enabled."
  resolution: "Enable SG reference in peering connection, or use CIDR-based rules instead."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-security-groups, describe-instances: GREEN — read-only"
  - "VPC Reachability Analyzer: GREEN — read-only path analysis"
  - "Add inbound security group rule: YELLOW — opens access, recoverable with revoke"
  - "Add outbound security group rule: YELLOW — opens egress, recoverable with revoke"
  - "Modify existing security group rule: YELLOW — changes access scope, recoverable"
  - "Delete security group: RED — permanently removes all rules, may break dependent resources"
```

## Escalation Conditions
- Security group is shared across multiple production instances or services
- Fix requires opening access from 0.0.0.0/0 or broad CIDR ranges
- Security group is managed by IaC (CloudFormation/Terraform) and manual changes would cause drift
- Cross-VPC security group reference requires peering configuration changes
- Security group has reached maximum rule limit (60 inbound + 60 outbound default)

## Data Sensitivity
- HIGH: describe-security-groups (reveals all network access rules, allowed CIDRs, referenced SG IDs)
- HIGH: describe-instances (reveals instance IPs, IAM roles, key pairs)
- MEDIUM: VPC flow logs (reveals traffic patterns and rejected connections)

## Prohibited Actions
- NEVER suggest opening 0.0.0.0/0 on any port to fix connectivity issues
- NEVER suggest removing all outbound rules to troubleshoot (breaks return traffic for stateful connections)
- NEVER suggest deleting a security group that is attached to running instances
- NEVER suggest using IP-based rules when SG-to-SG references are appropriate within the same VPC

## Phase 3 — Rollback
- If inbound rule was added: revoke with `aws ec2 revoke-security-group-ingress --group-id <sg> --protocol <proto> --port <port> --cidr <cidr>`
- If outbound rule was added: revoke with `aws ec2 revoke-security-group-egress`
- If security group was modified via IaC: run IaC plan/apply to restore desired state
- If wrong security group was attached to instance: modify instance attribute to restore original SG list

## Output Format

```yaml
root_cause: "Security group — <missing_inbound|missing_outbound|wrong_source|cross_vpc_sg_ref>"
evidence:
  - type: security_group_rules
    content: "<current rules and the gap>"
severity: MEDIUM
mitigation:
  immediate: "Add the required security group rule"
  long_term: "Document required ports, use IaC for SG management"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"

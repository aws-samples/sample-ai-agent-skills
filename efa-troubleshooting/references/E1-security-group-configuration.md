---
title: "E1 — Security Group Configuration"
description: "Diagnose EFA security group misconfigurations"
status: active
severity: HIGH
triggers: ["efa traffic blocked", "security group efa", "cannot communicate between nodes", "os-bypass blocked"]
owner: devops-agent
objective: "Resolve security group misconfigurations that block EFA OS-bypass traffic"
context: "EFA requires a security group with ALL traffic allowed from itself (self-referencing rule). Standard port-based rules are insufficient for OS-bypass."
---

## Phase 1 — Triage

MUST:
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[0].{Inbound:IpPermissions,Outbound:IpPermissionsEgress}'`
- Verify self-referencing rule exists: look for inbound rule with IpProtocol=-1 and source=same security group
- Confirm all EFA instances use the same security group

SHOULD:
- Check if multiple security groups are attached and if any deny EFA traffic
- Verify outbound rules allow all traffic (default allows all outbound)

MAY:
- Use VPC Flow Logs to identify blocked traffic: `aws ec2 describe-flow-logs --filter "Name=resource-id,Values=<vpc-id>"`

## Phase 2 — Remediate

MUST:
- Add self-referencing inbound rule: `aws ec2 authorize-security-group-ingress --group-id <sg-id> --protocol -1 --source-group <sg-id>`
- Verify the rule was applied: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Ensure all EFA instances are in this security group

SHOULD:
- Create a dedicated EFA security group for clarity
- Document the self-referencing rule requirement in runbooks

MAY:
- Use security group tags to identify EFA-specific groups

## Safety Ratings

```
safety_ratings:
  - "ec2 describe-security-groups: GREEN — read-only security group query"
  - "ec2 authorize-security-group-ingress: YELLOW — opens network access"
  - "ec2 describe-flow-logs: GREEN — read-only flow log query"
  - "Creating dedicated EFA security group: GREEN — new resource, no impact on existing"
```

## Escalation Conditions

- Organization SCPs prevent creating self-referencing security group rules
- Security team requires justification for all-traffic inbound rules
- Multiple security groups with conflicting rules are attached to instances
- VPC Flow Logs show blocked traffic but security group appears correct

## Data Sensitivity

- Security group rules: MEDIUM — reveals network access configuration
- VPC Flow Logs: HIGH — contain source/destination IPs and traffic patterns
- Security group IDs: LOW — identifiers only
- Instance security group associations: MEDIUM — reveals network topology

## Prohibited Actions

- NEVER add 0.0.0.0/0 as the source for the all-traffic rule — use self-referencing only
- NEVER remove existing security group rules without understanding their purpose
- NEVER modify security groups attached to production instances without change approval
- NEVER use NACL rules as a substitute for security group self-referencing

## Phase 3 — Rollback

1. Remove incorrectly added rules: `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol -1 --source-group <sg-id>`
2. If a dedicated EFA security group was created in error, remove from instances first, then delete
3. Restore original security group associations if they were changed
4. Verify EFA communication status after rollback
5. Document the security group configuration for compliance

## Common Issues

- symptoms: "fi_pingpong -p efa times out between two instances"
  diagnosis: "Security group does not have a self-referencing all-traffic inbound rule."
  resolution: "Add: aws ec2 authorize-security-group-ingress --group-id <sg-id> --protocol -1 --source-group <sg-id>."

- symptoms: "EFA works between some instances but not others"
  diagnosis: "Instances are in different security groups without cross-group all-traffic rules."
  resolution: "Ensure all EFA instances share the same security group with self-referencing rule."

## Output Format

```yaml
root_cause: "security_group — <specific_cause>"
evidence:
  - type: sg_configuration
    content: "<rule details>"
severity: HIGH
mitigation:
  immediate: "Add self-referencing all-traffic rule"
  long_term: "Create dedicated EFA security group in IaC"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"

---
title: "C1 — NACL Inbound Deny"
description: "Diagnose connectivity failures caused by NACL inbound deny rules"
status: active
severity: HIGH
triggers:
  - "Connection timed out"
  - "NACL blocking"
  - "REJECT in flow logs"
owner: devops-agent
objective: "Identify NACL inbound rules blocking traffic and restore access"
context: "NACLs are stateless subnet-level firewalls. Rules evaluated in order (lowest number first). First match wins. Default NACL allows all. Custom NACLs deny all by default. Inbound allow does NOT automatically allow outbound return traffic."
---

## Phase 1 — Triage

MUST:
- Identify the subnet's NACL: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`
- List inbound rules in order (lowest rule number first)
- Check if a DENY rule with a lower number blocks traffic before an ALLOW rule
- Verify if this is a custom NACL (default deny) or the default NACL (default allow)

SHOULD:
- Check flow logs for REJECT entries
- Verify the source IP/CIDR against each rule in order
- Check if the NACL was recently modified

MAY:
- Compare with NACLs on other subnets that work correctly
- Check CloudTrail for NACL rule changes

## Common Issues

- symptoms: "New subnet can't receive traffic, custom NACL"
  diagnosis: "Custom NACL denies all by default. No allow rules were added."
  resolution: "Add inbound allow rules for required traffic (and outbound for return traffic)."

- symptoms: "Deny rule 50 blocks IP range that includes the source, allow rule 100 exists"
  diagnosis: "NACL rules are evaluated in order. Rule 50 (deny) matches before rule 100 (allow)."
  resolution: "Add an allow rule with a number lower than the deny rule, or remove/modify the deny rule."

## Safety Ratings

```
safety_ratings:
  - "describe-network-acls: GREEN — read-only NACL rule inspection"
  - "describe-flow-logs: GREEN — read-only flow log query"
  - "create-network-acl-entry (add allow rule): YELLOW — adds NACL rule, recoverable by deleting the entry"
  - "replace-network-acl-entry (modify rule): YELLOW — changes existing rule, recoverable by replacing back"
  - "delete-network-acl-entry (remove deny rule): YELLOW — removes NACL rule, recoverable by re-adding"
  - "replace-network-acl-association: RED — changes subnet NACL association, can break all traffic to subnet"
```

## Escalation Conditions

- "Fix requires changing NACL rules (stateless, affects all traffic)"
- "Fix requires modifying production VPC route tables"
- "Multiple subnets or AZs affected"
- "Fix involves reordering NACL rules — affects evaluation order for all traffic"

## Data Sensitivity

- HIGH: NACL rules (expose network architecture and subnet-level access controls)
- HIGH: VPC flow logs (contain source/destination IPs and ports)
- MEDIUM: subnet CIDR ranges and NACL associations

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER modify NACL to deny all (can break connectivity for entire subnet)"

## Phase 3 — Rollback

- If a NACL rule was added via `create-network-acl-entry`: delete it with `aws ec2 delete-network-acl-entry --network-acl-id <nacl-id> --rule-number <num> --ingress`
- If a NACL rule was replaced via `replace-network-acl-entry`: replace it back with `aws ec2 replace-network-acl-entry --network-acl-id <nacl-id> --rule-number <num> --protocol <proto> --port-range From=<port>,To=<port> --cidr-block <cidr> --rule-action <allow|deny> --ingress`
- If a NACL rule was deleted: re-add it with `aws ec2 create-network-acl-entry` with the original parameters
- If NACL association was changed: revert with `aws ec2 replace-network-acl-association --association-id <id> --network-acl-id <original-nacl-id>`
- Document all NACL rules and associations before making changes

## Output Format

```yaml
root_cause: "nacl_inbound_deny — <rule detail>"
evidence:
  - type: network_acl
    content: "<NACL ID, rule number, and action>"
severity: HIGH
mitigation:
  immediate: "Add or reorder NACL rules to allow required traffic"
  long_term: "Document NACL rules, use IaC, prefer SGs over NACLs where possible"
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

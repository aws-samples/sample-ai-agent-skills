---
title: "Z1 — General VPC Troubleshooting"
description: "Catch-all runbook for VPC issues that don't match a specific category"
status: active
severity: MEDIUM
triggers:
  - "VPC networking issue"
  - "Something wrong with VPC"
  - "Network connectivity problem"
owner: devops-agent
objective: "Systematically investigate unclassified VPC networking issues"
context: "Use this runbook when the issue doesn't clearly match a specific category. Follow the systematic approach to narrow down the problem domain."
---

## Phase 1 — Triage

MUST:
- Identify source and destination resources (IPs, subnets, VPCs)
- Check route tables for both source and destination subnets
- Check security groups on both source and destination resources
- Check NACLs on both source and destination subnets
- Check VPC flow logs for ACCEPT/REJECT entries

SHOULD:
- Run Reachability Analyzer between source and destination
- Check if the issue is unidirectional or bidirectional
- Verify DNS resolution works
- Check if the issue started after a recent change (CloudTrail)

MAY:
- Check CloudWatch metrics for network-related anomalies
- Test connectivity from a known-good resource in the same subnet
- Check if the issue is intermittent or consistent

## Systematic Approach

1. Verify routing: Does a route exist for the destination?
2. Verify NACLs: Are both inbound AND outbound allowed (stateless)?
3. Verify security groups: Is the traffic allowed inbound on destination and outbound on source?
4. Verify the target is listening: Is the service running on the expected port?
5. Verify DNS: Does the hostname resolve to the correct IP?
6. Verify MTU: Are large packets being dropped?

## Safety Ratings

```
safety_ratings:
  - "describe-route-tables: GREEN — read-only route table inspection"
  - "describe-security-groups: GREEN — read-only SG rule inspection"
  - "describe-network-acls: GREEN — read-only NACL rule inspection"
  - "describe-flow-logs: GREEN — read-only flow log query"
  - "describe-vpc-attribute: GREEN — read-only VPC attribute check"
  - "create-route: YELLOW — adds route entry, recoverable by deleting"
  - "authorize-security-group-ingress: YELLOW — adds SG rule, recoverable by revoking"
  - "create-network-acl-entry: YELLOW — adds NACL rule, recoverable by deleting"
  - "modify-vpc-attribute: YELLOW — changes VPC setting, recoverable by reverting"
```

## Escalation Conditions

- "Fix requires modifying production VPC route tables"
- "Fix requires changing NACL rules (stateless, affects all traffic)"
- "Fix requires modifying Transit Gateway route tables"
- "Multiple subnets or AZs affected"
- "Root cause spans multiple networking layers (routing + SG + NACL)"

## Data Sensitivity

- HIGH: security group rules, NACL rules, route tables (expose network architecture)
- HIGH: VPC flow logs (contain source/destination IPs)
- MEDIUM: subnet CIDR ranges, VPC peering configurations

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"

## Phase 3 — Rollback

- If a route was added: delete it with `aws ec2 delete-route --route-table-id <rtb-id> --destination-cidr-block <cidr>`
- If an SG rule was added: revoke it with `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol <proto> --port <port> --cidr <cidr>`
- If a NACL rule was added: delete it with `aws ec2 delete-network-acl-entry --network-acl-id <nacl-id> --rule-number <num> --ingress` (or `--egress`)
- If a VPC attribute was changed: revert it with `aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --<attribute> <original-value>`
- Document all configurations across all layers before making any changes

## Output Format

```yaml
root_cause: "<category> — <detail>"
evidence:
  - type: <source>
    content: "<specific finding>"
severity: CRITICAL | HIGH | MEDIUM
mitigation:
  immediate: "<action>"
  long_term: "<prevention>"
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

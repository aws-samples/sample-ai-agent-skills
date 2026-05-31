---
title: "H1 — Gateway Endpoint Routing Issues"
description: "Diagnose S3/DynamoDB gateway endpoint connectivity failures"
status: active
severity: HIGH
triggers:
  - "Can't reach S3 from private subnet"
  - "DynamoDB timeout from VPC"
  - "Gateway endpoint not working"
owner: devops-agent
objective: "Fix gateway endpoint routing for S3 and DynamoDB access"
context: "Gateway endpoints (S3, DynamoDB) are free and add prefix list routes to route tables. They must be associated with the correct route tables. Traffic stays on AWS network. No DNS changes needed."
---

## Phase 1 — Triage

MUST:
- Check gateway endpoint exists: `aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id> Name=vpc-endpoint-type,Values=Gateway`
- Verify endpoint is associated with the correct route tables
- Check route table for prefix list entry (pl-xxx → vpce-xxx)
- Verify endpoint policy allows the required actions

SHOULD:
- Check if the endpoint is in the correct region (S3 endpoint is region-specific)
- Verify IAM role/policy on the instance allows S3/DynamoDB access
- Check if a restrictive endpoint policy is blocking specific buckets/tables

MAY:
- Test with a broad endpoint policy to confirm it's the issue
- Check if the application is using the correct regional S3 endpoint

## Common Issues

- symptoms: "S3 access works from some subnets but not others"
  diagnosis: "Gateway endpoint not associated with all required route tables."
  resolution: "Add the route table association for the affected subnets."

- symptoms: "Endpoint exists but S3 access denied"
  diagnosis: "Endpoint policy restricts access to specific buckets, or IAM policy doesn't allow S3."
  resolution: "Update endpoint policy or IAM policy to allow the required access."

## Safety Ratings

```
safety_ratings:
  - "describe-vpc-endpoints: GREEN — read-only endpoint inspection"
  - "describe-route-tables: GREEN — read-only route table inspection"
  - "describe-prefix-lists: GREEN — read-only prefix list check"
  - "modify-vpc-endpoint (add route table association): YELLOW — adds association, recoverable by removing"
  - "modify-vpc-endpoint (update endpoint policy): YELLOW — changes policy, recoverable by reverting"
  - "delete-vpc-endpoints: RED — deletes endpoint, breaks S3/DynamoDB access from private subnets"
```

## Escalation Conditions

- "Fix requires modifying gateway endpoint route table associations in production"
- "Fix requires updating endpoint policy — affects all traffic through the endpoint"
- "Multiple subnets affected by missing endpoint route table association"
- "Fix involves IAM policy changes in addition to endpoint policy"

## Data Sensitivity

- HIGH: endpoint policy (exposes which S3 buckets/DynamoDB tables are accessible)
- HIGH: route table associations (expose which subnets use the endpoint)
- MEDIUM: prefix list entries, VPC endpoint configuration

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER delete a gateway endpoint without confirming NAT-based fallback exists"

## Phase 3 — Rollback

- If a route table association was added: remove it with `aws ec2 modify-vpc-endpoint --vpc-endpoint-id <vpce-id> --remove-route-table-ids <rtb-id>`
- If the endpoint policy was updated: revert it with `aws ec2 modify-vpc-endpoint --vpc-endpoint-id <vpce-id> --policy-document <original-policy>`
- If the endpoint was deleted: recreate it with `aws ec2 create-vpc-endpoint --vpc-id <vpc-id> --service-name <service> --route-table-ids <rtb-ids>`
- Document endpoint configuration, route table associations, and policy before changes

## Output Format

```yaml
root_cause: "gateway_endpoint_routing — <detail>"
evidence:
  - type: vpc_endpoint
    content: "<endpoint configuration and route table associations>"
severity: HIGH
mitigation:
  immediate: "Associate endpoint with correct route tables or fix endpoint policy"
  long_term: "Use IaC to manage endpoint associations, use broad endpoint policies with IAM for fine-grained control"
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

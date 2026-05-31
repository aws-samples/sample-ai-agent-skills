---
title: "H2 — Serverless Endpoint Access"
description: "Diagnose and resolve Redshift Serverless endpoint connectivity issues"
status: active
severity: HIGH
triggers:
  - "Serverless endpoint"
  - "cannot connect Serverless"
  - "VPC endpoint"
  - "Serverless access"
owner: devops-agent
objective: "Fix connectivity to Redshift Serverless endpoints"
context: "Redshift Serverless workgroups have endpoints for client connections. Access can be public or VPC-based. VPC endpoints allow private connectivity. Security groups control network access. Cross-VPC access requires Redshift-managed VPC endpoints."
---

## Phase 1 — Triage

MUST:
- Check workgroup endpoint: `aws redshift-serverless get-workgroup --workgroup-name <workgroup> --query 'workgroup.endpoint'`
- Check workgroup VPC configuration: `aws redshift-serverless get-workgroup --workgroup-name <workgroup> --query 'workgroup.{subnets:subnetIds,securityGroups:securityGroupIds,publicAccess:publiclyAccessible}'`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[0].IpPermissions'`
- Test connectivity to endpoint on port 5439

SHOULD:
- Check if public access is enabled/disabled as intended
- Verify subnet route tables for proper routing
- Check for VPC endpoint access from other VPCs

MAY:
- Check DNS resolution of the endpoint
- Verify NACLs allow port 5439
- Check for cross-account endpoint access configuration

## Phase 2 — Remediate

MUST:
- If security group blocks access: add inbound rule for TCP 5439
- If public access needed: enable publicly accessible on workgroup
- If VPC routing issue: verify subnet route tables and NAT gateway
- If cross-VPC access needed: create Redshift-managed VPC endpoint

SHOULD:
- Restrict security groups to specific CIDR ranges
- Use VPC endpoints for private cross-VPC access
- Enable enhanced VPC routing for S3 traffic control

MAY:
- Set up VPC flow logs for connectivity debugging
- Implement connection monitoring
- Configure DNS for custom endpoint names

## Common Issues

- symptoms: "Connection timeout to Serverless endpoint"
  diagnosis: "Security group does not allow inbound TCP 5439 from client."
  resolution: "Add inbound rule for TCP 5439 from client CIDR or security group."

- symptoms: "Cannot connect from different VPC"
  diagnosis: "Serverless endpoint is VPC-bound. Cross-VPC requires managed VPC endpoint."
  resolution: "Create a Redshift-managed VPC endpoint for cross-VPC access."

## Output Format

```yaml
root_cause: "Serverless endpoint — <specific_cause>"
evidence:
  - type: endpoint_config
    content: "<workgroup endpoint and VPC configuration>"
severity: HIGH
mitigation:
  immediate: "Fix security group rules or VPC routing"
  long_term: "Use managed VPC endpoints for cross-VPC, restrict security groups"
```


## Safety Ratings
```
safety_ratings:
  - "Check workgroup endpoint: GREEN — read-only API call"
  - "Check VPC configuration: GREEN — read-only API call"
  - "Check security group rules: GREEN — read-only network inspection"
  - "Add security group inbound rule: YELLOW — changes network access"
  - "Enable public access: RED — exposes endpoint to internet"
  - "Create managed VPC endpoint: YELLOW — creates cross-VPC network path"
```

## Escalation Conditions
- Cluster serves production analytics
- Endpoint connectivity issues blocking production applications
- Public access change requires security review
- Cross-VPC endpoint needed for multi-VPC architecture
- Security group changes affect other services

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Endpoint URL: connection details"
    - "VPC configuration: network topology"
    - "Security group rules: access control configuration"
  handling: "Do not expose endpoint URLs, VPC topology, or security group details externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER open port 5439 to 0.0.0.0/0 without explicit security review
- NEVER enable public access on production workgroups without security approval
- NEVER remove security group rules without understanding impact on other services

## Phase 3 — Rollback
- If security group rules were added: remove the added rules
- If public access was enabled: disable public access on the workgroup
- If managed VPC endpoint was created: delete the VPC endpoint
- If enhanced VPC routing was changed: revert the setting on the workgroup

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
  - "NEVER suggest disabling encryption"

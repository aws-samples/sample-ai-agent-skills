---
title: "F1 — IAM/VPC Configuration"
description: "Diagnose and resolve IAM and VPC access issues for Redshift"
status: active
severity: HIGH
triggers:
  - "access denied"
  - "IAM role"
  - "VPC"
  - "security group"
  - "subnet"
  - "cannot connect"
  - "connection refused"
owner: devops-agent
objective: "Fix IAM permissions and VPC networking for Redshift access"
context: "Redshift clusters run in a VPC. Access requires proper security group rules (port 5439), subnet configuration, and IAM roles for S3/Glue/other service access. Enhanced VPC routing forces all COPY/UNLOAD traffic through the VPC."
---

## Phase 1 — Triage

MUST:
- Check cluster VPC configuration: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].{vpc:VpcId,subnet:ClusterSubnetGroupName,sg:VpcSecurityGroups,public:PubliclyAccessible,endpoint:Endpoint}'`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[0].IpPermissions'`
- Check IAM roles attached to cluster: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].IamRoles'`
- Test connectivity to cluster endpoint on port 5439

SHOULD:
- Check enhanced VPC routing status: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].EnhancedVpcRouting'`
- Verify subnet route tables have proper routes (NAT gateway for private subnets)
- Check if S3 VPC endpoint exists (required for COPY/UNLOAD with enhanced VPC routing)
- Verify IAM role trust policy allows Redshift to assume it

MAY:
- Check NACLs for port 5439 restrictions
- Verify DNS resolution of cluster endpoint
- Check for VPC peering or Transit Gateway routes if cross-VPC access

## Phase 2 — Remediate

MUST:
- If security group blocks port 5439: add inbound rule for TCP 5439 from client CIDR
- If IAM role missing: attach role to cluster: `aws redshift modify-cluster-iam-roles --cluster-identifier <cluster-id> --add-iam-roles <role-arn>`
- If enhanced VPC routing with no S3 endpoint: create S3 VPC endpoint or NAT gateway
- If publicly accessible but should be private (or vice versa): modify cluster accessibility

SHOULD:
- Use VPC endpoints for S3 and Glue when enhanced VPC routing is enabled
- Restrict security group to specific CIDR ranges, not 0.0.0.0/0
- Use IAM roles instead of access keys for S3 access from Redshift

MAY:
- Enable audit logging for connection tracking
- Implement VPC flow logs for network debugging
- Use Redshift-managed VPC endpoints for cross-VPC access

## Common Issues

- symptoms: "Cannot connect to Redshift cluster from application"
  diagnosis: "Security group does not allow inbound TCP 5439 from client IP."
  resolution: "Add inbound rule: TCP 5439 from client CIDR or security group."

- symptoms: "COPY from S3 fails with timeout when enhanced VPC routing is enabled"
  diagnosis: "No route to S3 from VPC — missing S3 VPC endpoint or NAT gateway."
  resolution: "Create S3 gateway VPC endpoint in the cluster's VPC, or add NAT gateway."

## Output Format

```yaml
root_cause: "IAM/VPC issue — <specific_cause>"
evidence:
  - type: vpc_config
    content: "<security group rules and VPC configuration>"
severity: HIGH
mitigation:
  immediate: "Fix security group rules, IAM role, or VPC routing"
  long_term: "Implement least-privilege security groups, use VPC endpoints, enable audit logging"
```


## Safety Ratings
```
safety_ratings:
  - "Check cluster VPC configuration: GREEN — read-only API call"
  - "Check security group rules: GREEN — read-only network inspection"
  - "Check IAM roles: GREEN — read-only IAM inspection"
  - "Add security group inbound rule: YELLOW — changes network access"
  - "Attach IAM role to cluster: YELLOW — grants new service access"
  - "Modify publicly accessible setting: RED — changes cluster exposure to internet"
  - "Create S3 VPC endpoint: YELLOW — changes VPC routing for S3 traffic"
```

## Escalation Conditions
- Cluster serves production analytics
- Security group changes affect multiple services
- IAM role changes impact cross-service access
- Enhanced VPC routing requires VPC endpoint creation
- Public accessibility change requires security review

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "VPC configuration: network topology"
    - "Security group rules: access control configuration"
    - "IAM role ARNs and policies: permission boundaries"
    - "Cluster endpoint: connection details"
    - "User credentials: database access information"
  handling: "Do not expose VPC topology, security group rules, or cluster endpoints externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER open port 5439 to 0.0.0.0/0 without explicit security review
- NEVER attach IAM roles with overly broad permissions (e.g., AdministratorAccess)
- NEVER make a production cluster publicly accessible without security approval

## Phase 3 — Rollback
- If security group rules were added: remove the added inbound/outbound rules
- If IAM role was attached: detach the role from the cluster
- If publicly accessible was changed: revert the setting
- If S3 VPC endpoint was created: delete the endpoint (COPY/UNLOAD may fail if enhanced VPC routing is on)
- If enhanced VPC routing was enabled: disable it (requires cluster modification)

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

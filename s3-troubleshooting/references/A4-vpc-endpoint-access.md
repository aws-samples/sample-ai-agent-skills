---
title: "A4 — VPC Endpoint S3 Access"
description: "Diagnose S3 access failures when using VPC gateway endpoints"
status: active
severity: MEDIUM
triggers:
  - "VPC endpoint S3 access denied"
  - "Cannot access S3 from VPC"
  - "vpce policy"
  - "S3 gateway endpoint"
owner: devops-agent
objective: "Identify and fix VPC endpoint configuration issues blocking S3 access"
context: "S3 Gateway endpoints route S3 traffic through the AWS network instead of the internet. Access requires correct endpoint policy, route table association, and DNS resolution. The endpoint policy acts as an additional policy layer — it does not grant access alone but can restrict it."
---

## Phase 1 — Triage

MUST:
- Identify the VPC endpoint: `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.s3`
- Check the endpoint policy: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <vpce-id> --query 'VpcEndpoints[0].PolicyDocument'`
- Verify route table association: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <vpce-id> --query 'VpcEndpoints[0].RouteTableIds'`
- Check the route table for the S3 prefix list: `aws ec2 describe-route-tables --route-table-ids <rtb-id>`

SHOULD:
- Verify DNS resolution from the instance: the S3 endpoint should resolve to private IPs
- Check if the bucket policy restricts to the VPC endpoint: `aws:sourceVpce` condition
- Verify the endpoint is in the same region as the bucket

MAY:
- Check VPC flow logs for dropped traffic
- Verify security groups and NACLs are not blocking traffic (gateway endpoints bypass these, but interface endpoints do not)

## Phase 2 — Remediate

MUST:
- Fix the endpoint policy to allow the required S3 actions and buckets
- Associate the endpoint with the correct route tables
- Ensure the bucket policy allows access from the VPC endpoint if using `aws:sourceVpce` conditions

SHOULD:
- Use a full-access endpoint policy and control access via IAM and bucket policies instead
- Verify all subnets that need S3 access have their route tables associated

MAY:
- Add `aws:sourceVpce` condition to the bucket policy to restrict access to the VPC endpoint only
- Consider S3 Interface endpoints if you need DNS-based access from on-premises

## Common Issues

- symptoms: "S3 access works from some subnets but not others"
  diagnosis: "The VPC endpoint is not associated with all required route tables."
  resolution: "Associate the endpoint with route tables for all subnets that need S3 access."

- symptoms: "Access denied only for specific buckets"
  diagnosis: "The VPC endpoint policy restricts access to specific buckets."
  resolution: "Update the endpoint policy to include the required bucket ARNs."

- symptoms: "S3 access stopped after adding aws:sourceVpce to bucket policy"
  diagnosis: "Requests not going through the endpoint (wrong route table) are denied by the bucket policy condition."
  resolution: "Ensure all clients use the VPC endpoint, or adjust the bucket policy condition."

## Output Format

```yaml
root_cause: "vpc_endpoint_access — <specific_cause>"
evidence:
  - type: endpoint_policy
    content: "<endpoint policy document>"
  - type: route_table
    content: "<route table entries>"
severity: MEDIUM
mitigation:
  immediate: "Fix endpoint policy or route table association"
  long_term: "Document VPC endpoint requirements for all S3-dependent workloads"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟢 GREEN | Primarily diagnostic — uses describe-vpc-endpoints, describe-route-tables, get-bucket-policy. Remediation targets VPC endpoint policies and route tables, not bucket-level security controls. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Bucket contains sensitive/regulated data (PII, PHI, financial)
- Cross-account access changes are needed
- Encryption configuration changes affect multiple consumers

## Rollback
1. Before any bucket policy change: Save current policy with `aws s3api get-bucket-policy`
2. Before ACL changes: Save current ACL with `aws s3api get-bucket-acl`
3. After change: Verify access works without granting excessive permissions
4. If change causes issues: Restore the saved policy/ACL immediately
5. Cleanup: Remove any temporary access grants

## Data Sensitivity
| Command | Sensitivity | Handling |
|---------|------------|----------|
| `get-bucket-policy` | HIGH | Contains access rules — redact principals |
| `get-bucket-acl` | MEDIUM | Shows grantees — summarize |
| `get-public-access-block` | MEDIUM | Security posture — safe to include |
| `list-objects` | LOW | Object keys only — safe to include |

## Prohibited Actions
- NEVER suggest disabling S3 Block Public Access as a remediation
- NEVER suggest `"Principal": "*"` in bucket policy without Condition keys
- NEVER suggest removing bucket encryption to fix access issues
- NEVER suggest making a bucket public to resolve CORS or access issues
- NEVER suggest `s3:*` in any IAM or bucket policy fix
- ALWAYS use least-privilege: grant only the specific S3 action needed
- ALWAYS check both account-level AND bucket-level Block Public Access

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "get-bucket-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-bucket-acl"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-public-access-block"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling S3 Block Public Access"
  - "NEVER suggest Principal: * without Condition keys"
  - "NEVER suggest removing bucket encryption"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback

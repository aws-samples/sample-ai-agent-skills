---
title: "A1 — Cluster Creation Failures"
description: "Diagnose why Neptune cluster creation is failing"
status: active
severity: HIGH
triggers:
  - "cluster creation failed"
  - "cannot create neptune cluster"
  - "neptune create error"
  - "cluster FAILED"
owner: devops-agent
objective: "Identify the root cause of Neptune cluster creation failures"
context: "Neptune cluster creation can fail due to subnet group configuration, security group rules, IAM role issues, KMS key problems, engine version incompatibility, or service quota limits. Neptune is VPC-only — no public endpoint."
---

## Phase 1 — Triage

MUST:
- Describe the cluster: `aws neptune describe-db-clusters --db-cluster-identifier <cluster-id>`
- Check cluster Status for error state
- Verify subnet group: `aws neptune describe-db-subnet-groups --db-subnet-group-name <subnet-group>`
- Check security group allows port 8182: `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Check CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=rds.amazonaws.com --max-results 20`
- Verify IAM role for S3 access (if bulk loading needed): `aws iam get-role --role-name <neptune-role>`
- Check KMS key accessibility: `aws kms describe-key --key-id <key-id>`
- Verify engine version availability: `aws neptune describe-db-engine-versions --engine neptune`

MAY:
- Check service quotas: `aws service-quotas get-service-quota --service-code neptune --quota-code L-E3E2D6CF`
- Verify parameter group: `aws neptune describe-db-cluster-parameter-groups`
- Check for region-specific limitations

## Phase 2 — Remediate

MUST:
- Fix the identified root cause (subnet group, security group, IAM, KMS)
- Ensure subnet group has subnets in at least 2 AZs
- Verify security group allows inbound TCP 8182

SHOULD:
- Use a dedicated subnet group for Neptune clusters
- Configure appropriate instance class for the workload
- Tag clusters for cost tracking

MAY:
- Set up CloudWatch alarms for cluster health
- Use CloudFormation/Terraform for consistent deployments

## Common Issues

- symptoms: "Cluster creation fails with subnet group error"
  diagnosis: "Subnet group does not have subnets in at least 2 AZs."
  resolution: "Create a subnet group with private subnets in at least 2 AZs."

- symptoms: "IAM role error during creation"
  diagnosis: "The Neptune service role does not have the required trust policy or S3 permissions."
  resolution: "Ensure the role has rds.amazonaws.com as trusted entity and s3:GetObject permissions."

- symptoms: "Engine version not available"
  diagnosis: "The requested engine version is not available in the selected region."
  resolution: "Use aws neptune describe-db-engine-versions to find available versions."

## Output Format

```yaml
root_cause: "cluster_creation — <specific_cause>"
evidence:
  - type: cluster
    content: "<describe-db-clusters output>"
  - type: subnet_group
    content: "<subnet group configuration>"
  - type: security_group
    content: "<security group rules>"
severity: HIGH
mitigation:
  immediate: "Fix configuration and retry cluster creation"
  long_term: "Use IaC templates for consistent cluster deployments"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — creation failures do not expose existing data
- Service Disruption Risk: LOW — creation failures do not affect existing clusters
- Compliance Impact: MEDIUM — misconfigured clusters may not meet security requirements

## Escalation Conditions
- Cluster creation fails repeatedly with internal service errors
- Creation blocked by AWS Organizations SCP policies
- Subnet group or VPC configuration prevents cluster creation
- IAM authentication setup fails during cluster creation
- Encryption configuration issues block cluster creation

## Data Sensitivity
- VPC and subnet configurations — INTERNAL
- IAM role ARNs — INTERNAL
- KMS key IDs — CONFIDENTIAL
- Parameter group settings — INTERNAL
- CloudTrail creation events — INTERNAL
- Never expose database endpoint addresses externally

## Prohibited Actions
- Do NOT create Neptune clusters in public subnets
- Do NOT disable encryption at rest for production clusters
- Do NOT use default parameter groups for production workloads
- Do NOT create clusters without IAM authentication enabled
- Do NOT skip subnet group configuration across multiple AZs
- Do NOT create clusters without enabling audit logs

## Phase 3 — Rollback
- Delete failed cluster resources and associated subnet groups if no longer needed
- Remove orphaned IAM roles created for the failed cluster
- Clean up security groups created specifically for the failed cluster
- Remove CloudWatch log groups from failed creation attempts
- Document failure reasons before retrying cluster creation
- Verify no residual resources remain from the failed creation

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
  - "NEVER suggest disabling IAM authentication"
  - "NEVER suggest public access"
  - "NEVER suggest disabling encryption at rest"

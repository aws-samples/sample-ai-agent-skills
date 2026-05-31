---
title: "A1 — Cluster Creation Failures"
description: "Diagnose why DocumentDB cluster creation is failing"
status: active
severity: HIGH
triggers:
  - "cluster creation failed"
  - "cannot create cluster"
  - "docdb create error"
  - "cluster FAILED"
owner: devops-agent
objective: "Identify the root cause of DocumentDB cluster creation failures"
context: "DocumentDB cluster creation can fail due to subnet group configuration, security group rules, KMS key issues, parameter group errors, insufficient IAM permissions, or service quota limits."
---

## Phase 1 — Triage

MUST:
- Describe the cluster: `aws docdb describe-db-clusters --db-cluster-identifier <cluster-id>`
- Check cluster Status field for error state
- Verify subnet group exists and has subnets in multiple AZs: `aws docdb describe-db-subnet-groups --db-subnet-group-name <subnet-group>`
- Check security group allows port 27017: `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Check CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=rds.amazonaws.com --max-results 20`
- Verify KMS key is accessible: `aws kms describe-key --key-id <key-id>`
- Check parameter group validity: `aws docdb describe-db-cluster-parameter-groups`
- Verify service quotas: `aws service-quotas get-service-quota --service-code docdb --quota-code L-69F3C3A0`

MAY:
- Check VPC configuration and routing
- Verify engine version is valid for the region
- Check for ongoing maintenance events in the region

## Phase 2 — Remediate

MUST:
- Fix the identified root cause (subnet group, security group, KMS, parameter group)
- Ensure subnet group has subnets in at least 2 AZs
- Verify security group allows inbound TCP 27017

SHOULD:
- Use a dedicated subnet group for DocumentDB clusters
- Configure automatic backups with appropriate retention
- Tag clusters for cost tracking

MAY:
- Set up CloudWatch alarms for cluster health
- Use CloudFormation/Terraform for consistent deployments

## Common Issues

- symptoms: "Cluster creation fails with subnet group error"
  diagnosis: "Subnet group does not have subnets in at least 2 AZs."
  resolution: "Create a subnet group with subnets in at least 2 AZs in the same VPC."

- symptoms: "KMS key access denied during creation"
  diagnosis: "The DocumentDB service role cannot access the specified KMS key."
  resolution: "Update KMS key policy to grant rds.amazonaws.com kms:CreateGrant, kms:Decrypt, kms:GenerateDataKey."

- symptoms: "Instance fails to launch in the cluster"
  diagnosis: "The specified instance class is not available in the selected AZ or region."
  resolution: "Choose an available instance class. Check aws docdb describe-orderable-db-instance-options."

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


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Cluster creation involves KMS keys, credentials, and network configuration |
| Blast Radius | LOW — Creation failures don't affect existing clusters |
| Rollback Complexity | LOW — Delete failed cluster and recreate |
| Compliance Scope | HIGH — Database clusters must meet security and encryption requirements |

## Escalation Conditions

- Escalate to security if KMS key issues prevent encrypted cluster creation
- Escalate to network team if subnet or security group configuration blocks creation
- Escalate to AWS Support if cluster gets stuck in a failed state

## Data Sensitivity

- **Master credentials** are database admin secrets — store in Secrets Manager
- **KMS key** protects all data at rest — key access is critical
- **Subnet and security group** configurations reveal network architecture

## PROHIBITED Actions

- Do NOT create clusters without encryption at rest enabled
- Do NOT use default security groups for production clusters
- Do NOT store master credentials in plain text or version control
- Do NOT create clusters in single-AZ subnet groups for production

## Phase 3 — Rollback

- Delete the failed cluster: `aws docdb delete-db-cluster --db-cluster-identifier <id> --skip-final-snapshot`
- Fix the identified issue (subnet group, KMS key, security group)
- Recreate the cluster with corrected configuration

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
  - "NEVER suggest disabling TLS"
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"

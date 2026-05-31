---
title: "A1 — MSK Cluster Creation Failures"
description: "Diagnose and resolve MSK cluster creation failures"
status: active
severity: HIGH
triggers:
  - "cluster creation failed"
  - "create cluster error"
  - "cluster stuck creating"
  - "provisioning failed"
  - "CreateCluster"
owner: devops-agent
objective: "Identify why an MSK cluster failed to create and resolve the underlying issue"
context: "MSK cluster creation can fail due to insufficient VPC resources (subnets, ENIs, IPs), IAM permission issues, KMS key access, invalid configuration, or service limits. Creation typically takes 15-30 minutes. A cluster stuck in CREATING for over an hour likely has an issue. Failed clusters may need to be deleted and recreated."
---

## Phase 1 — Triage

MUST:
- Check cluster status: `aws kafka describe-cluster-v2 --cluster-arn <cluster-arn>`
- Check cluster state and error info: `aws kafka describe-cluster-v2 --cluster-arn <cluster-arn> --query 'ClusterInfo.{State:State,StateInfo:StateInfo}'`
- Check CloudTrail for creation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateCluster --max-results 10`
- Check VPC subnet availability: `aws ec2 describe-subnets --subnet-ids <subnet-1> <subnet-2> <subnet-3> --query 'Subnets[*].{SubnetId:SubnetId,AZ:AvailabilityZone,AvailableIPs:AvailableIpAddressCount,State:State}'`

SHOULD:
- Check ENI limits in the VPC: `aws ec2 describe-network-interfaces --filters Name=vpc-id,Values=<vpc-id> --query 'length(NetworkInterfaces)'`
- Check security group exists: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check KMS key access (if encryption specified): `aws kms describe-key --key-id <key-id>`
- Check service quotas: `aws service-quotas get-service-quota --service-code kafka --quota-code L-7A612F49`

MAY:
- Check IAM role permissions for MSK service-linked role
- Verify subnet route tables have required routes
- Check if subnets are in different AZs (required for multi-AZ)

## Phase 2 — Remediate

MUST:
- If insufficient IPs: use subnets with more available IP addresses (MSK needs ENIs per broker)
- If KMS access denied: grant `kms:CreateGrant`, `kms:DescribeKey`, `kms:Encrypt`, `kms:Decrypt`, `kms:GenerateDataKey` to the MSK service-linked role
- If invalid configuration: fix the broker configuration and retry
- If service limit reached: request a quota increase via Service Quotas

SHOULD:
- Use subnets in at least 2 AZs (3 AZs recommended for production)
- Ensure each subnet has at least 5 available IP addresses per broker
- Delete failed clusters before retrying to avoid resource leaks

MAY:
- Use a custom MSK configuration instead of the default
- Enable enhanced monitoring at creation time
- Tag the cluster for cost allocation

## Common Issues

- symptoms: "Cluster stuck in CREATING state for over an hour"
  diagnosis: "Insufficient ENIs or IP addresses in the specified subnets."
  resolution: "Delete the cluster, ensure subnets have sufficient IPs, and recreate."

- symptoms: "CreateCluster returns AccessDeniedException"
  diagnosis: "IAM principal lacks kafka:CreateCluster permission or KMS key access."
  resolution: "Add required IAM permissions. If using CMK, grant KMS permissions to the MSK service-linked role."

- symptoms: "Cluster creation fails with InvalidSubnet"
  diagnosis: "Subnets are in the same AZ or do not exist."
  resolution: "Specify subnets in at least 2 different AZs. Verify subnet IDs are correct."

## Output Format

```yaml
root_cause: "cluster_creation_failure — <specific_cause>"
evidence:
  - type: cluster_status
    content: "<cluster state and error info>"
  - type: cloudtrail
    content: "<creation event details>"
  - type: vpc_resources
    content: "<subnet and ENI availability>"
severity: HIGH
mitigation:
  immediate: "Resolve the blocking issue and recreate the cluster"
  long_term: "Pre-validate VPC resources and permissions before cluster creation"
```


## Safety Ratings
```
safety_ratings:
  - "Describe cluster status: GREEN — read-only API call, no state change"
  - "Check CloudTrail events: GREEN — read-only audit log query"
  - "Check VPC subnet availability: GREEN — read-only network inspection"
  - "Delete failed cluster: YELLOW — removes cluster resource, but failed clusters have no data"
  - "Recreate cluster with new configuration: YELLOW — provisions new infrastructure with cost implications"
  - "Request quota increase: GREEN — no immediate infrastructure change"
```

## Escalation Conditions
- Cluster serves production event streaming
- Cluster creation failure blocks a production deployment or migration
- Repeated creation failures suggest underlying VPC or account-level issues
- Fix requires coordination with networking team for VPC/subnet changes

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "VPC and subnet configuration: network topology exposure"
    - "KMS key ARN: encryption key identifier"
    - "Security group IDs: network access control identifiers"
    - "IAM role ARNs: permission boundary identifiers"
  handling: "Mask full ARNs in logs. Do not expose VPC CIDR ranges externally."
```

## Prohibited Actions
- NEVER suggest deleting topics with active consumers
- NEVER suggest reducing broker count
- NEVER delete a cluster that is in ACTIVE state without explicit confirmation and backup plan
- NEVER modify VPC security groups without understanding the blast radius on other services sharing the VPC

## Phase 3 — Rollback
- If cluster was created with wrong configuration: delete the misconfigured cluster and recreate with correct settings (no data to preserve on a new cluster)
- If security group rules were modified: revert inbound/outbound rules to previous state using saved security group snapshot
- If KMS key policy was changed: restore previous key policy from CloudTrail event history
- If subnet route tables were modified: revert route table entries to previous configuration
- If service quota increase was requested: no rollback needed — quota increases are non-destructive

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest removing authentication"
  - "NEVER suggest public access to brokers"

---
title: "A1 — Cluster Creation Failures"
description: "Diagnose and resolve AWS CloudHSM cluster creation failures"
status: active
severity: HIGH
triggers:
  - "cluster creation failed"
  - "CreateCluster error"
  - "CloudHSM cluster stuck"
  - "cluster initialization"
owner: devops-agent
objective: "Identify why CloudHSM cluster creation failed and successfully provision a new cluster"
context: "CloudHSM cluster creation can fail due to VPC/subnet misconfiguration, insufficient IAM permissions, service-linked role issues, or AWS service limits. Clusters require a VPC with private subnets across AZs and proper security group configuration to allow HSM communication on port 2223-2225."
---

## Phase 1 — Triage

MUST:
- Check cluster status: `aws cloudhsmv2 describe-clusters --query 'Clusters[*].{ID:ClusterId,State:State,StateMessage:StateMessage}'`
- Verify the VPC and subnet configuration: `aws ec2 describe-subnets --subnet-ids <subnet-id> --query 'Subnets[*].{ID:SubnetId,AZ:AvailabilityZone,VPC:VpcId,CIDR:CidrBlock}'`
- Check IAM permissions for the calling principal: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names cloudhsm:CreateCluster`
- Verify the CloudHSM service-linked role exists: `aws iam get-role --role-name AWSServiceRoleForCloudHSM`
- Check CloudTrail for CreateCluster errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateCluster --max-results 10`

SHOULD:
- Verify security group allows inbound TCP 2223-2225 from client instances: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check service quotas for CloudHSM clusters: `aws service-quotas get-service-quota --service-code cloudhsm --quota-code L-A]`
- Verify subnet has available IP addresses: `aws ec2 describe-subnets --subnet-ids <subnet-id> --query 'Subnets[0].AvailableIpAddressCount'`

MAY:
- Check if the VPC has DNS hostnames and DNS resolution enabled: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsHostnames`
- Verify no conflicting NACL rules on the subnet: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`
- Review AWS Health Dashboard for CloudHSM service issues

## Phase 2 — Remediate

MUST:
- If service-linked role is missing: `aws iam create-service-linked-role --aws-service-name cloudhsm.amazonaws.com`
- If security group is misconfigured: add inbound rules for TCP 2223-2225 from the client security group
- If subnet has no available IPs: choose a different subnet or expand the CIDR range
- If service quota reached: request a quota increase via AWS Service Quotas console

SHOULD:
- Ensure subnets are in different AZs for high availability
- Tag the cluster for identification: `aws cloudhsmv2 tag-resource --resource-id <cluster-id> --tag-list Key=Environment,Value=<env>`
- Verify the cluster transitions to UNINITIALIZED state after creation

MAY:
- Create the cluster in multiple AZs for production workloads
- Set up CloudWatch alarms for cluster state changes
- Document the VPC and subnet configuration for future reference

## Common Issues

- symptoms: "CreateCluster returns AccessDeniedException"
  diagnosis: "The IAM principal lacks cloudhsm:CreateCluster permission or the service-linked role cannot be created."
  resolution: "Attach the AWSCloudHSMFullAccess policy or add cloudhsm:CreateCluster and iam:CreateServiceLinkedRole permissions."

- symptoms: "Cluster stuck in CREATE_IN_PROGRESS for extended time"
  diagnosis: "Subnet or security group misconfiguration preventing HSM network setup."
  resolution: "Verify subnet has available IPs, security group allows TCP 2223-2225, and NACLs are not blocking traffic."

- symptoms: "CreateCluster fails with SubnetNotFound"
  diagnosis: "The specified subnet ID does not exist or is in a different region."
  resolution: "Verify the subnet ID and ensure the CLI region matches the subnet's region."

- symptoms: "Cluster creation fails with CloudHsmInternalException"
  diagnosis: "AWS-side issue during cluster provisioning."
  resolution: "Wait and retry. If persistent, check AWS Health Dashboard and contact AWS Support."

## Output Format

```yaml
root_cause: "Cluster creation failure — <specific_cause>"
evidence:
  - type: cluster_status
    content: "<describe-clusters output>"
  - type: cloudtrail_events
    content: "<relevant CreateCluster error events>"
severity: HIGH
mitigation:
  immediate: "Fix the identified configuration issue and retry cluster creation"
  long_term: "Implement IaC templates with validated VPC/subnet/SG configurations for CloudHSM"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 IAM and security group changes: YELLOW — state-changing but recoverable"
  - "Phase 2 cluster creation retry: YELLOW — creates new billable resources"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Cluster creation fails repeatedly with internal errors"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Issue involves VPC or networking changes affecting other services"
  - "Service quota increase is needed for production workloads"

## Data Sensitivity

data_sensitivity:
  - command: "describe-clusters"
    sensitivity: MEDIUM
    contains: "Cluster IDs, VPC/subnet details, and HSM configuration"
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Network access rules and IP ranges"
  - command: "cloudtrail lookup-events"
    sensitivity: HIGH
    contains: "API call details including principal ARNs and request parameters"

## Prohibited Actions

prohibited_actions:
  - "NEVER delete an active CloudHSM cluster without explicit confirmation"
  - "NEVER modify security groups to allow 0.0.0.0/0 access to HSM ports"
  - "NEVER share CloudHSM cluster credentials or crypto officer passwords"
  - "NEVER disable VPC flow logs during troubleshooting"

## Phase 3 — Rollback

- Document the current VPC, subnet, and security group state before making changes
- If security group changes cause connectivity issues, revert to the previous inbound/outbound rules
- If a service-linked role was created unnecessarily, it can be deleted after verifying no clusters depend on it
- If a cluster was created in error, delete it: `aws cloudhsmv2 delete-cluster --cluster-id <cluster-id>`
- Verify all resources return to their pre-change state after rollback

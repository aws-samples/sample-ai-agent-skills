---
title: "A2 — HSM Provisioning Errors"
description: "Diagnose and resolve AWS CloudHSM individual HSM provisioning failures"
status: active
severity: HIGH
triggers:
  - "HSM provisioning failed"
  - "CreateHsm error"
  - "HSM degraded"
  - "HSM not active"
owner: devops-agent
objective: "Identify why an HSM instance failed to provision and restore the cluster to a healthy state"
context: "After a cluster is created, individual HSMs must be provisioned into subnets. HSM provisioning can fail due to subnet capacity, AZ availability, ENI limits, or service limits. Each HSM gets an ENI in the specified subnet. Production clusters should have at least 2 HSMs across different AZs."
---

## Phase 1 — Triage

MUST:
- Check HSM status within the cluster: `aws cloudhsmv2 describe-clusters --filters clusterIds=<cluster-id> --query 'Clusters[0].Hsms[*].{ID:HsmId,State:State,AZ:AvailabilityZone,ENI:EniId,IP:EniIp}'`
- Verify the target subnet has available IPs: `aws ec2 describe-subnets --subnet-ids <subnet-id> --query 'Subnets[0].AvailableIpAddressCount'`
- Check CloudTrail for CreateHsm errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateHsm --max-results 10`
- Verify ENI limits in the account: `aws service-quotas get-service-quota --service-code vpc --quota-code L-DF5E4CA3`

SHOULD:
- Check if the HSM ENI was created: `aws ec2 describe-network-interfaces --filters Name=description,Values="CloudHSM ENI" --query 'NetworkInterfaces[*].{ID:NetworkInterfaceId,Status:Status,SubnetId:SubnetId}'`
- Verify the cluster is in a valid state for adding HSMs: `aws cloudhsmv2 describe-clusters --filters clusterIds=<cluster-id> --query 'Clusters[0].State'`
- Check for AZ-specific service issues in AWS Health Dashboard

MAY:
- Review the cluster's backup history: `aws cloudhsmv2 describe-backups --filters clusterIds=<cluster-id>`
- Check if other HSMs in the cluster are healthy
- Verify no resource policies are blocking ENI creation in the subnet

## Phase 2 — Remediate

MUST:
- If subnet has no available IPs: provision the HSM in a different subnet within the same VPC
- If ENI limit reached: request a quota increase or clean up unused ENIs
- If HSM is stuck in CREATE_IN_PROGRESS: wait up to 20 minutes, then delete and retry
- If HSM shows DEGRADED: delete the HSM and create a new one: `aws cloudhsmv2 delete-hsm --cluster-id <cluster-id> --hsm-id <hsm-id>`

SHOULD:
- Provision HSMs in at least 2 different AZs for high availability
- Verify the new HSM synchronizes with existing cluster members
- Monitor HSM state transitions after provisioning

MAY:
- Add a third HSM for additional redundancy in production
- Set up CloudWatch Events for HSM state change notifications
- Update IaC templates to reflect the correct subnet and AZ configuration

## Common Issues

- symptoms: "CreateHsm returns CloudHsmInternalFailure"
  diagnosis: "AWS-side provisioning issue, possibly AZ capacity."
  resolution: "Retry in a different AZ. If persistent, contact AWS Support."

- symptoms: "HSM stuck in CREATE_IN_PROGRESS beyond 20 minutes"
  diagnosis: "ENI creation or HSM initialization stalled."
  resolution: "Delete the HSM and retry. Verify subnet and security group configuration."

- symptoms: "HSM shows DEGRADED state"
  diagnosis: "Hardware or software issue on the HSM instance."
  resolution: "Delete the degraded HSM and provision a new one. Data is preserved in the cluster."

- symptoms: "CreateHsm fails with SubnetMappingNotFound"
  diagnosis: "The cluster does not have a subnet mapping for the specified AZ."
  resolution: "Use a subnet in an AZ that was included during cluster creation, or create a new cluster with the desired AZ."

## Output Format

```yaml
root_cause: "HSM provisioning failure — <specific_cause>"
evidence:
  - type: hsm_status
    content: "<describe-clusters HSM details>"
  - type: cloudtrail_events
    content: "<relevant CreateHsm error events>"
severity: HIGH
mitigation:
  immediate: "Delete failed HSM and reprovision in a valid subnet/AZ"
  long_term: "Automate HSM health monitoring and implement multi-AZ provisioning in IaC"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 HSM deletion and recreation: RED — removes HSM instance (cluster data preserved)"
  - "Phase 2 ENI cleanup: YELLOW — state-changing but recoverable"

## Escalation Conditions

escalation_conditions:
  - "Multiple HSMs in the cluster show DEGRADED state simultaneously"
  - "HSM provisioning fails across all AZs in the region"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Cluster has only one HSM and it is failing"
  - "Issue involves potential key material availability concerns"

## Data Sensitivity

data_sensitivity:
  - command: "describe-clusters"
    sensitivity: HIGH
    contains: "HSM IDs, ENI IPs, and cluster configuration details"
  - command: "describe-network-interfaces"
    sensitivity: MEDIUM
    contains: "Network interface details and private IP addresses"
  - command: "cloudtrail lookup-events"
    sensitivity: HIGH
    contains: "API call details including principal ARNs"

## Prohibited Actions

prohibited_actions:
  - "NEVER delete the last HSM in a production cluster without confirming backups exist"
  - "NEVER modify HSM ENIs directly through EC2 APIs"
  - "NEVER expose HSM ENI IPs publicly"
  - "NEVER skip verifying cluster synchronization after adding a new HSM"

## Phase 3 — Rollback

- Before deleting a degraded HSM, verify a recent cluster backup exists: `aws cloudhsmv2 describe-backups --filters clusterIds=<cluster-id> --max-results 1`
- If a newly provisioned HSM causes cluster instability, delete it immediately
- If ENI changes were made, revert security group associations to their previous state
- Verify remaining HSMs in the cluster are in ACTIVE state after any changes
- Confirm client connectivity is restored after HSM replacement

---
title: "A1 — Cluster Launch Failures"
description: "Diagnose why an EMR cluster fails to launch"
status: active
severity: HIGH
triggers:
  - "cluster launch failed"
  - "cluster won't start"
  - "TERMINATED_WITH_ERRORS on launch"
  - "cluster creation error"
owner: devops-agent
objective: "Identify why EMR cluster launch is failing and resolve the issue"
context: "Cluster launch failures can stem from IAM role issues, EC2 capacity, subnet configuration, security group problems, AMI issues, or invalid configurations. The state change reason provides the initial clue."
---

## Phase 1 — Triage

MUST:
- Check cluster state and reason: `aws emr describe-cluster --cluster-id <id> --query 'Cluster.Status.{State:State,Reason:StateChangeReason}'`
- Verify EMR service role: `aws iam get-role --role-name EMR_DefaultRole`
- Verify EC2 instance profile: `aws iam get-instance-profile --instance-profile-name EMR_EC2_DefaultRole`
- Check subnet and VPC: `aws ec2 describe-subnets --subnet-ids <subnet-id>`

SHOULD:
- Check CloudTrail for RunJobFlow events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RunJobFlow --max-results 10`
- Verify instance type availability in the AZ
- Check security group configuration
- Verify S3 log bucket exists and is accessible

MAY:
- Check EC2 service quotas for the instance type
- Verify KMS key permissions if encryption is configured

## Phase 2 — Remediate

MUST:
- Fix IAM roles (service role and instance profile)
- Ensure subnet has available IPs and internet access (NAT or endpoints)
- Use available instance types in the target AZ

SHOULD:
- Enable termination protection for production clusters
- Configure multiple AZs in instance fleets for availability
- Test with a minimal cluster configuration first

MAY:
- Request EC2 capacity increases
- Create standardized cluster launch templates

## Common Issues

- symptoms: "TERMINATED_WITH_ERRORS — EC2 is not authorized"
  diagnosis: "EMR service role lacks EC2 permissions."
  resolution: "Attach AmazonEMRServicePolicy_v2 to the EMR service role."

- symptoms: "TERMINATED_WITH_ERRORS — insufficient capacity"
  diagnosis: "Instance type not available in the AZ."
  resolution: "Try a different instance type or AZ. Use instance fleets for flexibility."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix IAM roles (service role and instance profile) | YELLOW | IAM changes affect cluster permissions |
| Ensure subnet has available IPs and internet access | GREEN | Verification and network config — non-destructive |
| Use available instance types in target AZ | GREEN | Configuration guidance — no state change |
| Enable termination protection | GREEN | Adds protection — non-destructive |
| Request EC2 capacity increases | GREEN | Quota request — no immediate impact |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Launch failure blocks a time-sensitive data pipeline
- IAM role changes affect other services sharing the same roles

## Data Sensitivity

- **Classification: HIGH**
- Cluster configuration reveals instance types, subnet topology, and security group architecture
- IAM role policies expose data access patterns for HDFS and S3
- CloudTrail RunJobFlow events contain full cluster configuration including encryption settings
- S3 log bucket paths reveal organizational data lake structure

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest using overly permissive IAM policies (e.g., `*` resource) to resolve launch failures
- **NEVER** suggest launching clusters in public subnets without security justification

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Modified IAM service role | Revert IAM role policy to previous version |
| Modified EC2 instance profile | Revert instance profile policy to previous version |
| Changed subnet or VPC configuration | Revert network configuration changes |
| Enabled termination protection | Disable termination protection if not needed |

## Output Format

```yaml
root_cause: "launch_failure — <specific_cause>"
evidence:
  - type: state_change_reason
    content: "<cluster state change reason>"
  - type: iam_roles
    content: "<IAM role configuration>"
severity: HIGH
mitigation:
  immediate: "Fix the specific launch blocker"
  long_term: "Create validated cluster templates and automate pre-launch checks"
```

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
  - "NEVER suggest disabling encryption"
  - "NEVER suggest public subnet placement for clusters"
  - "NEVER suggest removing Kerberos authentication"

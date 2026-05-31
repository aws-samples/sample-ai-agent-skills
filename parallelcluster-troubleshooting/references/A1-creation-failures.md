---
title: "A1 — Cluster Creation Failures"
description: "Diagnose ParallelCluster creation failures"
status: active
severity: CRITICAL
triggers:
  - "cluster creation failed"
  - "CREATE_FAILED"
  - "pcluster create"
  - "stack failed"
owner: devops-agent
objective: "Identify and resolve the root cause of ParallelCluster creation failures"
context: "Cluster creation uses CloudFormation. Failures are typically due to IAM permissions, VPC configuration, AMI issues, instance type availability, or configuration errors. The CloudFormation stack events contain the actual error."
---

## Phase 1 — Triage

MUST:
- Check cluster status: `pcluster describe-cluster --cluster-name <cluster-name>`
- Get stack events: `pcluster get-cluster-stack-events --cluster-name <cluster-name>`
- Check CloudFormation stack: `aws cloudformation describe-stack-events --stack-name parallelcluster-<cluster-name> --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'`
- Review cluster configuration file for errors

SHOULD:
- Validate config before creation: `pcluster create-cluster --cluster-name <name> --cluster-configuration <config> --dryrun true`
- Check IAM permissions for the user/role creating the cluster
- Verify VPC and subnet exist and have proper configuration

MAY:
- Check service quotas for EC2 instance types
- Review CloudTrail for permission-related failures

## Phase 2 — Remediate

MUST:
- Fix the error identified in CloudFormation stack events
- If IAM: add required permissions (ec2, efs, fsx, s3, cloudformation, iam)
- If VPC: verify subnet has internet access or VPC endpoints
- If AMI: use official ParallelCluster AMI or build with pcluster build-image

SHOULD:
- Delete the failed cluster before retrying: `pcluster delete-cluster --cluster-name <cluster-name>`
- Use --dryrun to validate before creating
- Check instance type availability in the target AZ

MAY:
- Use --rollback-on-failure false for debugging (keeps failed resources)
- Review ParallelCluster release notes for known issues

## Common Issues

- symptoms: "CREATE_FAILED on HeadNode resource"
  diagnosis: "IAM permissions insufficient or AMI not compatible."
  resolution: "Check IAM policy. Use official ParallelCluster AMI."

- symptoms: "CREATE_FAILED on NetworkInterface or Subnet"
  diagnosis: "VPC/subnet misconfiguration or no internet access."
  resolution: "Verify subnet has NAT gateway route. Check security groups."

## Output Format

```yaml
root_cause: "creation_failure — <specific_cause>"
evidence:
  - type: stack_events
    content: "<CloudFormation failure event>"
  - type: cluster_config
    content: "<relevant configuration>"
severity: CRITICAL
mitigation:
  immediate: "Fix the identified error and recreate the cluster"
  long_term: "Implement config validation and IaC for cluster management"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — creation failures do not expose existing data
- Service Disruption Risk: LOW — creation failures do not affect existing clusters
- Compliance Impact: MEDIUM — cluster security configuration affects HPC workload compliance

## Escalation Conditions
- Cluster creation fails repeatedly with CloudFormation stack errors
- Creation blocked by IAM or SCP policies
- VPC or subnet configuration prevents cluster creation
- AMI issues prevent head node or compute node launch
- Creation fails with service quota limits

## Data Sensitivity
- CloudFormation stack parameters — INTERNAL (may contain CONFIDENTIAL data)
- IAM role configurations — INTERNAL
- VPC and subnet details — INTERNAL
- SSH key references — CONFIDENTIAL
- Never expose SSH private keys or cluster admin credentials

## Prohibited Actions
- Do NOT create clusters with overly permissive IAM roles
- Do NOT use default security groups for production HPC clusters
- Do NOT create clusters without encryption for sensitive workloads
- Do NOT share cluster SSH keys across teams
- Do NOT create clusters in public subnets without justification
- Do NOT bypass cluster configuration validation

## Phase 3 — Rollback
- Delete failed CloudFormation stacks and associated resources
- Clean up orphaned IAM roles and security groups
- Remove failed cluster configurations
- Document creation failure reasons before retrying
- Verify no residual resources remain from failed creation
- Update cluster configuration templates based on findings

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"

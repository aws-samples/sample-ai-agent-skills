---
title: "A1 — Cluster Creation Failures"
description: "Diagnose why MemoryDB cluster creation is failing"
status: active
severity: HIGH
triggers:
  - "cluster creation failed"
  - "cannot create cluster"
  - "memorydb create error"
  - "cluster FAILED"
owner: devops-agent
objective: "Identify the root cause of MemoryDB cluster creation failures"
context: "MemoryDB cluster creation can fail due to subnet group configuration, security group rules, ACL configuration, KMS key issues, node type availability, or service quota limits."
---

## Phase 1 — Triage

MUST:
- Describe the cluster: `aws memorydb describe-clusters --cluster-name <cluster-name>`
- Check cluster Status for error state
- Verify subnet group: `aws memorydb describe-subnet-groups --subnet-group-name <subnet-group>`
- Check security group allows port 6379: `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Check CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=memorydb.amazonaws.com --max-results 20`
- Verify ACL exists: `aws memorydb describe-acls --acl-name <acl-name>`
- Check KMS key accessibility: `aws kms describe-key --key-id <key-id>`
- Verify node type availability in the region

MAY:
- Check service quotas: `aws service-quotas get-service-quota --service-code memorydb --quota-code L-D62A8E22`
- Verify parameter group: `aws memorydb describe-parameter-groups`
- Check for region-specific limitations

## Phase 2 — Remediate

MUST:
- Fix the identified root cause (subnet group, security group, ACL, KMS)
- Ensure subnet group has subnets in at least 2 AZs (multi-AZ required)
- Verify ACL is properly configured with at least one user

SHOULD:
- Use a dedicated subnet group for MemoryDB clusters
- Configure appropriate shard count and replica count for the workload
- Tag clusters for cost tracking

MAY:
- Set up CloudWatch alarms for cluster health
- Use CloudFormation/Terraform for consistent deployments

## Common Issues

- symptoms: "Cluster creation fails with subnet group error"
  diagnosis: "Subnet group does not have subnets in at least 2 AZs (MemoryDB requires multi-AZ)."
  resolution: "Create a subnet group with subnets in at least 2 AZs."

- symptoms: "ACL validation error during creation"
  diagnosis: "The specified ACL does not exist or has no users associated."
  resolution: "Create the ACL and associate at least one user before cluster creation."

- symptoms: "Node type not available"
  diagnosis: "The requested node type is not available in the selected region or AZ."
  resolution: "Choose an available node type. Check aws memorydb describe-reserved-nodes-offerings for options."

## Output Format

```yaml
root_cause: "cluster_creation — <specific_cause>"
evidence:
  - type: cluster
    content: "<describe-clusters output>"
  - type: subnet_group
    content: "<subnet group configuration>"
  - type: acl
    content: "<ACL configuration>"
severity: HIGH
mitigation:
  immediate: "Fix configuration and retry cluster creation"
  long_term: "Use IaC templates for consistent cluster deployments"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete a MemoryDB cluster without verifying snapshot backups and data durability requirements
- **NEVER** modify ACL rules on production clusters without testing authentication changes in a staging environment
- **NEVER** force a failover on a production cluster during peak traffic without customer approval
- **NEVER** reduce the number of shards or replicas on a cluster serving production workloads without capacity analysis
- **NEVER** disable encryption in-transit or at-rest on clusters handling sensitive data

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest removing ACL authentication"
  - "NEVER suggest public subnet placement"

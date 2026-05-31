---
title: "A1 — Firewall Creation Failures"
description: "Diagnose Network Firewall creation failures"
status: active
severity: HIGH
triggers:
  - "firewall creation failed"
  - "cannot create firewall"
  - "CreateFirewall error"
owner: devops-agent
objective: "Identify why Network Firewall creation is failing and resolve the issue"
context: "Firewall creation requires a VPC, subnets (one per AZ), a firewall policy, and IAM permissions. Creation can fail due to subnet configuration issues, policy errors, service limits, or permission problems. Firewall subnets should be dedicated (no workloads)."
---

## Phase 1 — Triage

MUST:
- Check IAM permissions for network-firewall:CreateFirewall
- Verify VPC and subnets exist: `aws ec2 describe-subnets --subnet-ids <subnet-ids>`
- Verify firewall policy exists: `aws network-firewall describe-firewall-policy --firewall-policy-name <policy-name>`
- Check service quotas: `aws service-quotas get-service-quota --service-code network-firewall --quota-code L-DE9D3F8E`

SHOULD:
- Verify subnets are in different AZs (one subnet per AZ)
- Check CloudTrail for creation errors
- Verify subnets have available IP addresses

MAY:
- Check if the VPC has an internet gateway (required for some configurations)
- Verify no conflicting firewall exists in the same subnets

## Phase 2 — Remediate

MUST:
- Create firewall with valid configuration: `aws network-firewall create-firewall --firewall-name <name> --firewall-policy-arn <policy-arn> --vpc-id <vpc-id> --subnet-mappings SubnetId=<subnet-id>`
- Use dedicated subnets for firewall endpoints
- Ensure one subnet per AZ maximum

SHOULD:
- Tag firewall for management
- Enable delete protection for production firewalls

MAY:
- Use CloudFormation/Terraform for repeatable deployment
- Deploy in multiple AZs for high availability

## Common Issues

- symptoms: "Subnet already has a firewall endpoint"
  diagnosis: "Cannot have multiple firewall endpoints in the same subnet."
  resolution: "Use a different subnet or remove existing firewall from the subnet."

- symptoms: "Firewall policy not found"
  diagnosis: "Policy ARN incorrect or policy in different region."
  resolution: "Verify policy ARN. Create policy if it doesn't exist."

- symptoms: "Insufficient permissions"
  diagnosis: "IAM policy missing network-firewall:CreateFirewall."
  resolution: "Add Network Firewall permissions to IAM policy."

## Output Format

```yaml
root_cause: "creation_failure — <specific_cause>"
evidence:
  - type: vpc_subnets
    content: "<VPC and subnet configuration>"
  - type: policy_status
    content: "<firewall policy status>"
  - type: permissions
    content: "<IAM permissions>"
severity: HIGH
mitigation:
  immediate: "Fix the specific creation blocker"
  long_term: "Use IaC for firewall deployment"
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

- **NEVER** delete or modify firewall rules that are actively filtering production traffic without a tested rollback plan
- **NEVER** change route table associations for firewall endpoints without verifying traffic flow continuity
- **NEVER** disable TLS inspection on endpoints monitoring compliance-regulated traffic
- **NEVER** remove stateful rule groups from a firewall policy during active threat mitigation
- **NEVER** modify asymmetric routing configurations without confirming bidirectional traffic inspection

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
  - "NEVER suggest disabling stateful inspection"
  - "NEVER suggest removing all firewall rules"
  - "NEVER suggest bypassing TLS inspection without review"

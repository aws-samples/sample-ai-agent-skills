---
title: "E1 — VPC Endpoint Issues"
description: "Diagnose VPC endpoint connectivity for Managed Blockchain"
status: active
severity: HIGH
triggers: ["cannot connect", "endpoint error", "VPC access denied"]
owner: devops-agent
objective: "Identify and resolve VPC endpoint connectivity issues"
---

## Phase 1 — Triage

MUST:
- Verify VPC endpoint exists: `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.managedblockchain.<network-id>`
- Check endpoint status is available
- Verify security group allows inbound on required ports
- Check subnet configuration

SHOULD:
- Verify DNS resolution for endpoint
- Check route table configuration
- Verify endpoint policy allows access

## Phase 2 — Remediate

MUST:
- Create VPC endpoint for Managed Blockchain
- Fix security group rules (allow 443 for HTTPS)
- Ensure endpoint is in correct subnets

SHOULD:
- Enable private DNS for the endpoint
- Document VPC endpoint configuration

## Output Format

```yaml
root_cause: "vpc_endpoint — <specific_cause>"
severity: HIGH
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

- **NEVER** delete a member node from a Hyperledger Fabric network without verifying quorum and endorsement policies
- **NEVER** modify chaincode endorsement policies on production channels without multi-party approval
- **NEVER** remove VPC endpoints that are actively used for blockchain node communication
- **NEVER** force-delete a network or member with active channels and committed transactions
- **NEVER** alter Ethereum node sync configurations during active transaction processing

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"

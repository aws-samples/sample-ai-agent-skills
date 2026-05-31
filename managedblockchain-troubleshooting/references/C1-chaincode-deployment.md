---
title: "C1 — Chaincode Deployment Issues"
description: "Diagnose Fabric chaincode deployment failures"
status: active
severity: HIGH
triggers: ["chaincode install failed", "instantiate error", "chaincode not found"]
owner: devops-agent
objective: "Identify and resolve chaincode deployment failures"
context: "Deployment fails when: package invalid, peer not accessible, endorsement policy wrong, or lifecycle steps skipped."
---

## Phase 1 — Triage

MUST:
- Verify chaincode package is valid (Go, Node.js, or Java)
- Check if chaincode is installed on all required peers
- Verify chaincode is approved for the organization
- Check if chaincode is committed to the channel

SHOULD:
- Review chaincode logs in CloudWatch
- Verify endorsement policy is achievable
- Check peer node status

## Phase 2 — Remediate

MUST:
- Follow complete lifecycle: package → install → approve → commit
- Install on all endorsing peers
- Fix endorsement policy to match available peers

SHOULD:
- Test chaincode locally before deployment
- Document chaincode version and endorsement policy

## Output Format

```yaml
root_cause: "chaincode_deployment — <specific_cause>"
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

---
title: "G2 — Encryption Issues"
description: "Diagnose encryption at rest and in transit issues on MemoryDB"
status: active
severity: HIGH
triggers:
  - "encryption error"
  - "KMS key"
  - "encryption at rest"
  - "TLS encryption"
owner: devops-agent
objective: "Identify and resolve encryption configuration issues on MemoryDB"
context: "MemoryDB encrypts data at rest using KMS keys and always requires TLS for in-transit encryption. KMS key is set at cluster creation and is immutable. TLS cannot be disabled."
---

## Phase 1 — Triage

MUST:
- Check cluster encryption: `aws memorydb describe-clusters --cluster-name <cluster-name>`
- Verify KMS key status: `aws kms describe-key --key-id <key-id>`
- Verify TLS is working (always enabled): `redis-cli --tls -h <endpoint> PING`
- Check KMS key policy

SHOULD:
- Verify KMS key grants: `aws kms list-grants --key-id <key-id>`
- Check for KMS-related errors in CloudTrail
- Verify key rotation status
- Check snapshot encryption

MAY:
- Review cross-account KMS access if applicable
- Check KMS key usage metrics
- Verify compliance requirements

## Phase 2 — Remediate

MUST:
- Re-enable KMS key if disabled
- Cancel key deletion if pending
- Ensure KMS key policy grants MemoryDB access

SHOULD:
- Enable automatic KMS key rotation
- Use customer-managed keys for production
- Document encryption configuration

MAY:
- Set up KMS key monitoring
- Implement encryption compliance checks

## Common Issues

- symptoms: "Cluster inaccessible after KMS key disabled"
  diagnosis: "KMS key used for encryption at rest was disabled."
  resolution: "Re-enable the KMS key. Cluster should recover automatically."

- symptoms: "Cannot change encryption key"
  diagnosis: "Encryption key is immutable after cluster creation."
  resolution: "Create a new cluster with the desired KMS key. Restore from snapshot."

- symptoms: "TLS connection issues"
  diagnosis: "Client TLS configuration is incorrect (MemoryDB always requires TLS)."
  resolution: "Ensure client uses TLS 1.2+. Use --tls flag with redis-cli."

## Output Format

```yaml
root_cause: "encryption — <specific_cause>"
evidence:
  - type: encryption_config
    content: "<cluster encryption configuration>"
  - type: kms_key
    content: "<key status and policy>"
  - type: tls
    content: "<TLS connectivity status>"
severity: HIGH
mitigation:
  immediate: "Fix KMS key status or client TLS configuration"
  long_term: "Implement encryption key management best practices"
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

---
title: "G1 — ACL Management Issues"
description: "Diagnose ACL configuration and user management problems"
status: active
severity: MEDIUM
triggers:
  - "ACL management"
  - "create user"
  - "access string"
  - "ACL configuration"
  - "user management"
owner: devops-agent
objective: "Identify and resolve ACL management and configuration issues"
context: "MemoryDB ACLs control authentication and authorization. Each ACL contains users with passwords and access strings. ACLs are associated with clusters. Proper ACL management is critical for security and access control."
---

## Phase 1 — Triage

MUST:
- List ACLs: `aws memorydb describe-acls`
- List users: `aws memorydb describe-users`
- Check ACL-cluster associations: `aws memorydb describe-clusters --cluster-name <cluster-name>`
- Verify user access strings are correct

SHOULD:
- Check if the default user has open access (security risk)
- Verify user status is active
- Review access string syntax for correctness
- Check for unused or orphaned ACLs

MAY:
- Review ACL modification history via CloudTrail
- Check for password rotation compliance
- Verify access string covers required commands and keys

## Phase 2 — Remediate

MUST:
- Restrict the default user in production (remove open-access)
- Create application-specific users with least-privilege access strings
- Associate the correct ACL with each cluster

SHOULD:
- Implement password rotation procedures
- Document ACL and user management policies
- Use separate users for different applications

MAY:
- Automate ACL management with IaC
- Implement ACL auditing
- Set up alerts for ACL changes

## Common Issues

- symptoms: "Default user has unrestricted access"
  diagnosis: "The default user (open-access ACL) allows all commands and keys without authentication."
  resolution: "Create a custom ACL with restricted users. Associate it with the cluster. Disable default user access."

- symptoms: "Cannot modify ACL associated with a cluster"
  diagnosis: "ACL modifications propagate to associated clusters and may cause brief disruption."
  resolution: "Modify ACL during maintenance windows. Changes propagate automatically to clusters."

- symptoms: "Access string syntax error"
  diagnosis: "Access string uses incorrect Redis ACL syntax."
  resolution: "Use correct syntax: on ~* +@all (full access) or on ~key:* +@read (restricted). Check Redis ACL documentation."

## Output Format

```yaml
root_cause: "acl_management — <specific_cause>"
evidence:
  - type: acls
    content: "<ACL list and configuration>"
  - type: users
    content: "<user details and access strings>"
  - type: clusters
    content: "<ACL-cluster associations>"
severity: MEDIUM
mitigation:
  immediate: "Fix ACL configuration and user access"
  long_term: "Implement ACL management best practices and auditing"
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

---
title: "C2 — ACL Authentication Issues"
description: "Diagnose ACL authentication and authorization failures on MemoryDB"
status: active
severity: HIGH
triggers:
  - "ACL error"
  - "authentication failed"
  - "NOAUTH"
  - "NOPERM"
  - "access denied"
  - "user password"
owner: devops-agent
objective: "Identify and resolve ACL authentication and authorization issues"
context: "MemoryDB uses ACLs for authentication and authorization. Each ACL has users with passwords and access strings. ACLs are associated with clusters. Issues include incorrect credentials, missing users, restrictive access strings, or ACL-cluster association problems."
---

## Phase 1 — Triage

MUST:
- Check cluster ACL association: `aws memorydb describe-clusters --cluster-name <cluster-name>`
- List ACL details: `aws memorydb describe-acls --acl-name <acl-name>`
- List users in the ACL: `aws memorydb describe-users`
- Verify user status is active

SHOULD:
- Check user access string for command and key restrictions
- Verify the password is correct (test with redis-cli)
- Check if the default user is being used (security risk)
- Review ACL modification history via CloudTrail

MAY:
- Check for multiple ACLs and their cluster associations
- Review user password rotation history
- Verify access string syntax

## Phase 2 — Remediate

MUST:
- Create or fix user credentials
- Ensure user is associated with the correct ACL
- Verify ACL is associated with the cluster
- Update access string to allow required commands and keys

SHOULD:
- Restrict the default user (remove open-access)
- Use specific access strings following least-privilege
- Implement password rotation procedures

MAY:
- Set up multiple users for different application roles
- Implement ACL management automation
- Enable audit logging for authentication events

## Common Issues

- symptoms: "NOAUTH — Authentication required"
  diagnosis: "Client did not provide username/password or ACL requires authentication."
  resolution: "Provide ACL username and password in the connection: AUTH username password."

- symptoms: "NOPERM — user has no permission for command"
  diagnosis: "User's access string does not allow the specific Redis command."
  resolution: "Update user access string to include the required commands: +@all or specific +command."

- symptoms: "WRONGPASS — invalid username-password pair"
  diagnosis: "Password is incorrect or user does not exist in the ACL."
  resolution: "Verify username and password. Reset password with aws memorydb update-user if needed."

## Output Format

```yaml
root_cause: "acl_authentication — <specific_cause>"
evidence:
  - type: acl
    content: "<ACL configuration and users>"
  - type: user
    content: "<user status and access string>"
  - type: cluster
    content: "<cluster ACL association>"
severity: HIGH
mitigation:
  immediate: "Fix user credentials or access string"
  long_term: "Implement ACL management best practices and rotation"
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

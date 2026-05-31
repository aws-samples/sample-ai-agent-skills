---
title: "G1 — Authentication Issues"
description: "Diagnose broker authentication and user management problems"
status: active
severity: HIGH
triggers:
  - "authentication failed"
  - "login failed"
  - "user not found"
  - "password incorrect"
  - "access denied"
owner: devops-agent
objective: "Identify and resolve authentication issues"
context: "Authentication issues include: incorrect credentials, user not created, user permissions insufficient, LDAP integration issues (ActiveMQ), or console access not enabled."
---

## Phase 1 — Triage

MUST:
- List broker users: `aws mq list-users --broker-id <broker-id>`
- Check specific user: `aws mq describe-user --broker-id <broker-id> --username <username>`
- Verify user has console access if trying to access web console
- Check broker logs for authentication failures

SHOULD:
- Verify password meets complexity requirements
- Check if user groups are configured correctly
- Verify LDAP configuration if using LDAP (ActiveMQ)

MAY:
- Check for pending user changes (require reboot)
- Review audit logs for authentication events

## Phase 2 — Remediate

MUST:
- Create missing user: `aws mq create-user --broker-id <broker-id> --username <user> --password <pass> --console-access`
- Update password: `aws mq update-user --broker-id <broker-id> --username <user> --password <new-pass>`
- Reboot broker after user changes

SHOULD:
- Enable console access for admin users
- Configure user groups for authorization
- Use strong passwords

MAY:
- Implement LDAP integration for centralized auth (ActiveMQ)
- Set up audit logging for authentication events

## Output Format

```yaml
root_cause: "authentication_issue — <specific_cause>"
evidence:
  - type: user_config
    content: "<user details>"
  - type: auth_logs
    content: "<authentication failure logs>"
severity: HIGH
mitigation:
  immediate: "Fix user credentials or create missing user"
  long_term: "Implement centralized authentication and audit logging"
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

- **NEVER** force-reboot a production broker during active message processing without draining connections
- **NEVER** delete a broker with persistent messages that have not been consumed or backed up
- **NEVER** modify authentication credentials on production brokers without coordinating with all connected clients
- **NEVER** change network-of-brokers topology during peak message throughput periods
- **NEVER** downgrade broker engine versions or instance types on production brokers without capacity testing

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

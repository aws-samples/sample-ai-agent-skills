---
title: "H1 — Authentication Issues"
description: "Diagnose authentication and authorization problems on DocumentDB"
status: active
severity: HIGH
triggers:
  - "authentication failed"
  - "auth error"
  - "login failed"
  - "unauthorized"
  - "user credentials"
owner: devops-agent
objective: "Identify and resolve authentication and authorization issues on DocumentDB"
context: "DocumentDB uses SCRAM-SHA-1 authentication with users stored in the admin database. The master user is created with the cluster. Additional users can be created with specific database roles. Authentication issues include incorrect credentials, missing users, and insufficient role permissions."
---

## Phase 1 — Triage

MUST:
- Verify cluster is accessible: `aws docdb describe-db-clusters --db-cluster-identifier <cluster-id>`
- Check the master username from cluster description
- Test authentication: `mongo --tls --host <endpoint> --username <user> --password <pass> --authenticationDatabase admin --eval "db.runCommand({connectionStatus: 1})"`
- Verify the user exists and has appropriate roles

SHOULD:
- Check if the password was recently rotated
- Verify authentication database is specified (must be admin for DocumentDB)
- Check for IAM authentication configuration if using IAM auth
- Review user roles and permissions

MAY:
- Check audit logs for authentication failures
- Verify Secrets Manager integration if using automatic rotation
- Review connection string authentication parameters

## Phase 2 — Remediate

MUST:
- Reset master user password if forgotten: `aws docdb modify-db-cluster --db-cluster-identifier <cluster-id> --master-user-password <new-password>`
- Create missing users with appropriate roles
- Ensure authenticationDatabase=admin in connection strings

SHOULD:
- Use Secrets Manager for credential management and rotation
- Implement least-privilege roles for application users
- Document user and role management procedures

MAY:
- Enable audit logging for authentication events
- Implement IAM authentication for enhanced security

## Common Issues

- symptoms: "Authentication failed — incorrect credentials"
  diagnosis: "Password is wrong or user does not exist in the admin database."
  resolution: "Verify credentials. Reset master password via modify-db-cluster if needed."

- symptoms: "User authenticated but unauthorized for operation"
  diagnosis: "User lacks the required role for the database or collection."
  resolution: "Grant appropriate roles: db.grantRolesToUser('user', [{role: 'readWrite', db: 'mydb'}])."

- symptoms: "Authentication works with mongo shell but not from application"
  diagnosis: "Application driver is not specifying authenticationDatabase=admin or using wrong auth mechanism."
  resolution: "Add authSource=admin and authMechanism=SCRAM-SHA-1 to the connection string."

## Output Format

```yaml
root_cause: "authentication — <specific_cause>"
evidence:
  - type: cluster
    content: "<cluster master username and auth configuration>"
  - type: auth_test
    content: "<authentication test results>"
  - type: user_roles
    content: "<user roles and permissions>"
severity: HIGH
mitigation:
  immediate: "Fix credentials or user roles"
  long_term: "Implement Secrets Manager and least-privilege roles"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — Authentication controls access to all database data |
| Blast Radius | HIGH — Credential changes affect all applications connecting to the cluster |
| Rollback Complexity | MEDIUM — Password resets are immediate but require application updates |
| Compliance Scope | CRITICAL — Authentication is a fundamental security control |

## Escalation Conditions

- Escalate IMMEDIATELY to security if unauthorized access is suspected
- Escalate to operations if credential rotation breaks application connectivity
- Escalate to compliance if authentication doesn't meet minimum standards

## Data Sensitivity

- **Database credentials** are authentication secrets — highest sensitivity
- **User roles and permissions** define data access scope
- **Connection strings** may contain credentials — never log or share

## PROHIBITED Actions

- Do NOT store database credentials in plain text, code, or version control
- Do NOT use the master user for application connections — create dedicated users
- Do NOT grant readWriteAnyDatabase role to application users
- Do NOT reset the master password without coordinating with all application teams
- Do NOT share credentials via email or chat

## Phase 3 — Rollback

- If password was changed incorrectly, reset again: `aws docdb modify-db-cluster --db-cluster-identifier <id> --master-user-password <new-password>`
- Update all application connection strings with the new password
- If user roles were changed incorrectly, restore previous role assignments
- Rotate credentials in Secrets Manager if they were exposed

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
  - "NEVER suggest disabling TLS"
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"

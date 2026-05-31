---
title: "C1 — Client Connectivity"
description: "Diagnose WorkMail client connectivity issues"
status: active
severity: MEDIUM
triggers: ["cannot connect", "Outlook not connecting", "web client error", "IMAP failed", "client timeout"]
owner: devops-agent
objective: "Resolve email client connectivity issues including Outlook, web client, and IMAP configuration"
context: "WorkMail supports web client, Outlook via IMAP, and mobile via EAS. Access control rules may block specific clients or IPs."
---

## Phase 1 — Triage

MUST:
- Verify user account is enabled: `aws workmail describe-user --organization-id <org-id> --user-id <user-id>`
- Check access control rules: `aws workmail list-access-control-rules --organization-id <org-id>`
- Verify the client is using correct server settings (endpoint, port, encryption)
- Test web client access to isolate client-specific vs service-wide issues

SHOULD:
- Check if the user's IP is blocked by access control rules
- Verify IMAP settings: server `imap.mail.<region>.awsapps.com`, port 993, SSL
- Check if the user's password has expired in the directory

MAY:
- Review CloudTrail for authentication failures
- Check if corporate firewall or proxy blocks WorkMail endpoints
- Test connectivity from a different network

## Phase 2 — Remediate

MUST:
- Fix client configuration with correct server settings
- Remove or modify access control rules blocking the user
- Reset user password if expired: through directory or WorkMail console

SHOULD:
- Update client to the latest version
- Configure firewall rules to allow WorkMail endpoints (ports 993, 443)

MAY:
- Set up application-specific passwords for clients that don't support modern auth
- Configure auto-discover for Outlook clients

## Safety Ratings

```
safety_ratings:
  - "workmail describe-user: GREEN — read-only user inspection"
  - "workmail list-access-control-rules: GREEN — read-only rule listing"
  - "Password reset: YELLOW — changes user authentication credentials"
  - "Access control rule changes: RED — affects who can connect to WorkMail"
```

## Escalation Conditions

- All clients fail to connect suggesting service-wide issue
- Access control rules cannot be modified due to admin restrictions
- Client connectivity fails despite correct configuration and no blocking rules
- Corporate network blocks WorkMail endpoints and firewall changes require approval

## Data Sensitivity

- `aws workmail describe-user`: MEDIUM — reveals user details
- `aws workmail list-access-control-rules`: MEDIUM — reveals access policies
- Client configuration: MEDIUM — contains server endpoints and authentication details
- CloudTrail authentication events: MEDIUM — reveals login attempts and source IPs

## Prohibited Actions

- NEVER disable all access control rules to troubleshoot connectivity
- NEVER share user passwords in troubleshooting logs
- NEVER configure clients to use unencrypted connections (no SSL/TLS)
- NEVER modify access control rules without understanding the security impact

## Phase 3 — Rollback

1. If access control rules were modified, restore original rules
2. If user password was reset, notify the user and ensure they can set a new password
3. If client configuration was changed, document the original settings for revert
4. If firewall rules were modified, revert if the change was temporary
5. Verify all affected users can connect after changes

## Common Issues

- symptoms: "Outlook cannot connect — authentication failed"
  diagnosis: "Incorrect password, expired password, or IMAP settings wrong."
  resolution: "Verify IMAP settings: imap.mail.<region>.awsapps.com, port 993, SSL. Reset password if expired."

- symptoms: "Client connects from office but not from home"
  diagnosis: "Access control rule restricts connections to specific IP ranges."
  resolution: "Review access control rules. Add the user's home IP range or remove IP restriction."

## Output Format

```yaml
root_cause: "client_connectivity — <specific_cause>"
evidence:
  - type: connectivity_error
    content: "<client settings, access rules, user status>"
severity: MEDIUM
mitigation:
  immediate: "Fix client configuration or access rules"
  long_term: "Document client setup procedures"
```

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

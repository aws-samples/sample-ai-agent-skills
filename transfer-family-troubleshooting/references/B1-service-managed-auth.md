---
title: "B1 — Service-Managed Authentication Failures"
description: "Diagnose authentication failures when using Transfer Family service-managed identity provider"
status: active
severity: HIGH
triggers:
  - "authentication failed"
  - "login failed"
  - "SSH key rejected"
  - "service managed auth"
  - "cannot log in"
  - "access denied on login"
owner: devops-agent
objective: "Identify why users cannot authenticate using service-managed SSH keys"
context: "Service-managed authentication stores SSH public keys directly on the Transfer Family user resource. Only SSH key authentication is supported (no passwords). Users must present a private key matching one of their stored public keys. Common failures include incorrect key format, wrong key pair, or user misconfiguration."
---

## Phase 1 — Triage

MUST:
- Verify server uses service-managed auth: `aws transfer describe-server --server-id <server-id> --query 'Server.IdentityProviderType'`
- Check user exists and has SSH keys: `aws transfer describe-user --server-id <server-id> --user-name <username> --query 'User.SshPublicKeys'`
- Verify the user's IAM role is valid: `aws iam get-role --role-name <role-name>`
- Check the role trust policy includes transfer.amazonaws.com: `aws iam get-role --role-name <role-name> --query 'Role.AssumeRolePolicyDocument'`
- Check structured logs for auth errors: `aws logs filter-log-events --log-group-name /aws/transfer/<server-id> --filter-pattern "\"authentication\""`

SHOULD:
- Verify the SSH public key format is correct (must start with ssh-rsa, ecdsa-sha2-nistp256, etc.)
- Check if the user's home directory exists (S3 prefix or EFS path)
- Verify the IAM role has permissions to access the storage backend

MAY:
- Test with a different SSH key pair to isolate key-specific issues
- Check if the user was recently created and propagation is complete

## Phase 2 — Remediate

MUST:
- Ensure SSH public key is in the correct format (OpenSSH format, single line, no comments after the key)
- Verify the private key on the client matches the public key stored in Transfer Family
- Fix IAM role trust policy to trust transfer.amazonaws.com

SHOULD:
- Rotate SSH keys if compromise is suspected
- Use ECDSA or ED25519 keys instead of RSA for better security
- Ensure the home directory path exists in the storage backend

MAY:
- Implement key rotation procedures with scheduled key updates
- Enable structured logging for ongoing authentication monitoring

## Common Issues

- symptoms: "Permission denied (publickey) when connecting via SFTP"
  diagnosis: "Private key on client does not match any public key stored for the user."
  resolution: "Re-import the correct public key or generate a new key pair."

- symptoms: "User authenticates but immediately disconnected"
  diagnosis: "IAM role trust policy does not trust transfer.amazonaws.com."
  resolution: "Update the role trust policy to allow sts:AssumeRole by transfer.amazonaws.com."

- symptoms: "SSH key import fails with invalid format"
  diagnosis: "Key is in PEM/PKCS8 format instead of OpenSSH format, or has extra whitespace/newlines."
  resolution: "Convert key to OpenSSH format: ssh-keygen -i -f key.pem > key.pub"

## Output Format

```yaml
root_cause: "service_managed_auth — <specific_cause>"
evidence:
  - type: identity_provider
    content: "<identity provider type>"
  - type: ssh_keys
    content: "<user SSH key configuration>"
  - type: iam_role
    content: "<role trust policy and permissions>"
severity: HIGH
mitigation:
  immediate: "Fix SSH key or IAM role configuration"
  long_term: "Implement key rotation and structured logging"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Transferred files may contain any type of sensitive data
- SSH keys and credentials enable server access
- Directory mappings reveal S3/EFS data organization
- Classification: CONFIDENTIAL - file transfer content and access credentials

## Prohibited Actions

- NEVER modify production server endpoints during active file transfers
- NEVER change authentication providers without user notification
- NEVER delete SSH host keys without coordinating with all SFTP clients
- NEVER modify directory mappings that could expose unauthorized S3/EFS paths

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest disabling SFTP encryption"
  - "NEVER suggest FTP protocol for production"
  - "NEVER suggest overly broad IAM roles for transfer users"

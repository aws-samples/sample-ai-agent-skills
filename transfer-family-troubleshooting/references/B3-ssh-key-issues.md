---
title: "B3 — SSH Key Authentication Issues"
description: "Diagnose SSH key format, mismatch, and management problems"
status: active
severity: MEDIUM
triggers:
  - "SSH key"
  - "public key"
  - "private key"
  - "key format"
  - "key mismatch"
  - "publickey denied"
owner: devops-agent
objective: "Resolve SSH key authentication issues including format errors, key mismatches, and key management problems"
context: "Transfer Family supports RSA, ECDSA, and ED25519 SSH keys in OpenSSH format. Keys must be imported in the correct format. Common issues include PEM vs OpenSSH format confusion, key pair mismatches, and key rotation problems."
---

## Phase 1 — Triage

MUST:
- List user's SSH keys: `aws transfer describe-user --server-id <server-id> --user-name <username> --query 'User.SshPublicKeys[*].{KeyId:SshPublicKeyId,DateImported:DateImported,Body:SshPublicKeyBody}'`
- Verify key format starts with a valid prefix (ssh-rsa, ecdsa-sha2-nistp256, ecdsa-sha2-nistp384, ecdsa-sha2-nistp521, ssh-ed25519)
- Check client-side key: verify the private key file permissions (should be 600 or 400)
- Test connection with verbose output: `sftp -vvv -i <private-key> <username>@<endpoint>`
- Check structured logs for key-specific errors: `aws logs filter-log-events --log-group-name /aws/transfer/<server-id> --filter-pattern "\"publickey\""`

SHOULD:
- Verify the public key fingerprint matches: `ssh-keygen -lf <public-key-file>`
- Check if multiple keys are stored and which one the client is presenting
- Verify the key algorithm is supported by the server's security policy

MAY:
- Check if the key was recently rotated and the old key was removed
- Verify no extra whitespace or line breaks in the stored public key

## Phase 2 — Remediate

MUST:
- Import keys in OpenSSH format (single line: `ssh-rsa AAAA... user@host`)
- Ensure client private key matches the stored public key
- Set correct file permissions on the client private key (chmod 600)

SHOULD:
- Use ECDSA or ED25519 keys for better security and performance
- Remove old/unused SSH keys from user configurations
- Document key rotation procedures

MAY:
- Implement automated key rotation using Lambda and Transfer Family APIs
- Use SSH certificates instead of raw keys for enterprise environments

## Common Issues

- symptoms: "Key import fails with 'Invalid SSH public key' error"
  diagnosis: "Key is in PEM format, has line breaks, or includes extra metadata."
  resolution: "Convert to OpenSSH format: ssh-keygen -y -f private.pem > public.pub"

- symptoms: "Client presents key but server rejects it"
  diagnosis: "Key algorithm not in the server's security policy, or key pair mismatch."
  resolution: "Check security policy supported algorithms. Regenerate key pair if mismatch suspected."

- symptoms: "Authentication works for one user but not another with the same key type"
  diagnosis: "Second user has the public key imported with extra whitespace or wrong format."
  resolution: "Delete and re-import the public key, ensuring clean OpenSSH format."

## Output Format

```yaml
root_cause: "ssh_key_issue — <specific_cause>"
evidence:
  - type: ssh_keys
    content: "<stored public keys and formats>"
  - type: client_key
    content: "<client key details and fingerprint>"
  - type: security_policy
    content: "<supported key algorithms>"
severity: MEDIUM
mitigation:
  immediate: "Fix key format or re-import correct key"
  long_term: "Implement key rotation and standardize on modern key algorithms"
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

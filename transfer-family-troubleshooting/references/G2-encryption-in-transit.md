---
title: "G2 — Encryption in Transit Issues"
description: "Diagnose TLS/SSH encryption, security policy, and cipher negotiation problems"
status: active
severity: HIGH
triggers:
  - "encryption"
  - "TLS"
  - "SSL"
  - "cipher"
  - "security policy"
  - "handshake failed"
  - "protocol version"
owner: devops-agent
objective: "Resolve encryption and security policy issues affecting Transfer Family connections"
context: "Transfer Family uses security policies to control which cryptographic algorithms are allowed. SFTP uses SSH encryption. FTPS uses TLS encryption. FTP is unencrypted. Security policies define allowed ciphers, key exchange algorithms, and MAC algorithms. Clients must support at least one algorithm in the server's policy to connect."
---

## Phase 1 — Triage

MUST:
- Check the server's security policy: `aws transfer describe-server --server-id <server-id> --query 'Server.SecurityPolicyName'`
- Get security policy details: `aws transfer describe-security-policy --security-policy-name <policy-name>`
- For FTPS, check the ACM certificate: `aws transfer describe-server --server-id <server-id> --query 'Server.Certificate'`
- Verify certificate validity: `aws acm describe-certificate --certificate-arn <cert-arn>`
- Test connection with verbose output to see cipher negotiation: `sftp -vvv <username>@<endpoint>` or `openssl s_client -connect <endpoint>:21 -starttls ftp`

SHOULD:
- Check if the client supports the ciphers in the server's security policy
- Verify the FTPS certificate domain matches the server endpoint or custom hostname
- Check if the security policy was recently changed (CloudTrail)
- Compare the security policy with the client's TLS/SSH capabilities

MAY:
- Test with different security policies to identify the minimum compatible policy
- Check if intermediate CAs are properly chained in the ACM certificate
- Verify the client's TLS/SSH library version supports modern algorithms

## Phase 2 — Remediate

MUST:
- Use a security policy compatible with the client's cipher support
- For FTPS, ensure a valid ACM certificate is associated and not expired
- Ensure the certificate covers the server's endpoint hostname (or custom domain)

SHOULD:
- Use the most restrictive security policy that all clients support (TransferSecurityPolicy-2024-01 preferred)
- Upgrade legacy clients to support modern ciphers rather than weakening the security policy
- Document the minimum client requirements for each security policy

MAY:
- Implement certificate rotation monitoring with ACM auto-renewal
- Set up CloudWatch alarms for certificate expiration
- Consider using SFTP instead of FTPS to simplify certificate management

## Common Issues

- symptoms: "FTPS connection fails with 'no shared cipher' or TLS handshake error"
  diagnosis: "Server security policy requires ciphers the client does not support."
  resolution: "Use a less restrictive security policy or upgrade the client. Check describe-security-policy for supported ciphers."

- symptoms: "FTPS certificate warning — hostname mismatch"
  diagnosis: "ACM certificate does not cover the Transfer Family endpoint hostname or custom domain."
  resolution: "Issue a new ACM certificate with the correct domain name (server endpoint or custom hostname)."

- symptoms: "SFTP connection fails after security policy change"
  diagnosis: "New security policy removed the SSH cipher or key exchange algorithm the client was using."
  resolution: "Revert to the previous security policy or upgrade the client's SSH library."

## Output Format

```yaml
root_cause: "encryption — <specific_cause>"
evidence:
  - type: security_policy
    content: "<policy name and supported algorithms>"
  - type: certificate
    content: "<ACM certificate status and domains>"
  - type: client_negotiation
    content: "<cipher negotiation details>"
severity: HIGH
mitigation:
  immediate: "Fix security policy or certificate configuration"
  long_term: "Standardize on modern ciphers and implement certificate monitoring"
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

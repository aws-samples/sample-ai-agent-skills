---
title: "A3 — Transfer Family Protocol Configuration Issues"
description: "Diagnose protocol misconfiguration for SFTP, FTPS, and FTP"
status: active
severity: MEDIUM
triggers:
  - "protocol not working"
  - "SFTP connection refused"
  - "FTPS handshake failed"
  - "FTP not enabled"
  - "protocol configuration"
  - "multi-protocol"
owner: devops-agent
objective: "Identify and resolve protocol-specific configuration issues on Transfer Family servers"
context: "Transfer Family supports SFTP (SSH File Transfer Protocol, port 22), FTPS (FTP over TLS, ports 21 + 8192-8200), and FTP (unencrypted, ports 21 + 8192-8200). A single server can support multiple protocols, but each has different port, security, and authentication requirements."
---

## Phase 1 — Triage

MUST:
- Check enabled protocols: `aws transfer describe-server --server-id <server-id> --query 'Server.Protocols'`
- Check protocol details: `aws transfer describe-server --server-id <server-id> --query 'Server.ProtocolDetails'`
- Check security policy: `aws transfer describe-server --server-id <server-id> --query 'Server.SecurityPolicyName'`
- Get security policy details: `aws transfer describe-security-policy --security-policy-name <policy-name>`
- Verify security groups allow all required ports for enabled protocols

SHOULD:
- Test connectivity on each protocol port: `nc -zv <endpoint> 22` (SFTP), `nc -zv <endpoint> 21` (FTP/FTPS)
- Check if FTPS certificate is configured: `aws transfer describe-server --server-id <server-id> --query 'Server.Certificate'`
- Verify the ACM certificate is valid and in the same region: `aws acm describe-certificate --certificate-arn <cert-arn>`

MAY:
- Check if AS2 protocol is enabled for B2B transfers
- Review protocol-specific CloudWatch metrics

## Phase 2 — Remediate

MUST:
- Open all required ports in security groups: port 22 for SFTP, ports 21 + 8192-8200 for FTP/FTPS
- For FTPS, ensure a valid ACM certificate is associated with the server
- Ensure the security policy supports ciphers compatible with client software

SHOULD:
- Use SFTP or FTPS instead of unencrypted FTP for production workloads
- Use the most restrictive security policy that clients can support (TransferSecurityPolicy-2024-01)
- Test protocol connectivity from the client network before going live

MAY:
- Enable multiple protocols on a single server to support diverse client requirements
- Configure AS2 for EDI/B2B partner integrations

## Common Issues

- symptoms: "FTPS connection fails with TLS handshake error"
  diagnosis: "No ACM certificate associated with the server, or certificate is expired/invalid."
  resolution: "Associate a valid ACM certificate. Ensure the certificate domain matches the server endpoint."

- symptoms: "SFTP works but FTP does not on the same server"
  diagnosis: "Security group only allows port 22. FTP requires port 21 and 8192-8200."
  resolution: "Add inbound rules for port 21 and port range 8192-8200 in the security group."

- symptoms: "Client cannot negotiate cipher with server"
  diagnosis: "Server security policy uses ciphers not supported by the client."
  resolution: "Check client cipher support. Use a less restrictive security policy if needed, or upgrade the client."

## Output Format

```yaml
root_cause: "protocol_config — <specific_cause>"
evidence:
  - type: protocols
    content: "<enabled protocols>"
  - type: security_policy
    content: "<policy name and supported ciphers>"
  - type: certificate
    content: "<ACM certificate status>"
severity: MEDIUM
mitigation:
  immediate: "Fix protocol configuration, ports, or certificate"
  long_term: "Standardize on SFTP or FTPS with restrictive security policies"
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

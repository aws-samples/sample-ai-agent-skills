---
title: "G1 — Host Key Management Issues"
description: "Diagnose SFTP host key mismatch warnings and key management problems"
status: active
severity: HIGH
triggers:
  - "host key"
  - "fingerprint"
  - "host key mismatch"
  - "man-in-the-middle"
  - "known_hosts"
  - "server identity"
owner: devops-agent
objective: "Resolve host key management issues including key migration, rotation, and client-side warnings"
context: "SFTP clients cache the server's host key fingerprint on first connection. If the host key changes (server replacement, key rotation, migration), clients display a warning and may refuse to connect. Transfer Family allows importing custom host keys and maintaining multiple keys per server. The most recently added active key is presented to clients."
---

## Phase 1 — Triage

MUST:
- List server host keys: `aws transfer list-host-keys --server-id <server-id>`
- Get host key details and fingerprint: `aws transfer describe-host-key --server-id <server-id> --host-key-id <host-key-id>`
- Check which key is currently active (most recently added active key is used)
- Verify the client's known_hosts file contains the correct fingerprint
- Check structured logs for host key-related connection failures

SHOULD:
- Compare the server's current fingerprint with the client's cached fingerprint
- Check if the server was recently replaced or migrated
- Verify the host key algorithm matches what the client expects (RSA, ECDSA, ED25519)
- Check the security policy for supported host key algorithms

MAY:
- Check CloudTrail for recent ImportHostKey or DeleteHostKey events
- Verify no automated process is rotating host keys unexpectedly

## Phase 2 — Remediate

MUST:
- For server migration: export the host key from the old server and import to the new server using `aws transfer import-host-key`
- For legitimate key change: notify all clients to update their known_hosts file
- For compromised key: rotate the host key and require all clients to accept the new fingerprint

SHOULD:
- Document the current host key fingerprint for all servers
- Implement a host key distribution process for managed clients
- Use a consistent host key algorithm across all servers

MAY:
- Automate host key backup as part of server provisioning
- Implement host key fingerprint verification in client automation scripts

## Common Issues

- symptoms: "SFTP client shows 'WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED'"
  diagnosis: "Server was replaced or host key was rotated. Client's known_hosts has the old fingerprint."
  resolution: "Import the old host key to the new server, or update the client's known_hosts with the new fingerprint."

- symptoms: "Automated SFTP scripts fail with 'Host key verification failed'"
  diagnosis: "Script uses StrictHostKeyChecking=yes and the host key changed."
  resolution: "Update the known_hosts file in the script's SSH config, or import the original host key to the server."

- symptoms: "Different clients see different host key fingerprints"
  diagnosis: "Server has multiple host keys with different algorithms. Clients negotiate different key types."
  resolution: "This is normal behavior. Each client negotiates the best algorithm it supports. Document all active fingerprints."

## Output Format

```yaml
root_cause: "host_key — <specific_cause>"
evidence:
  - type: host_keys
    content: "<server host key list and fingerprints>"
  - type: client_known_hosts
    content: "<client cached fingerprint>"
  - type: cloudtrail
    content: "<recent host key changes>"
severity: HIGH
mitigation:
  immediate: "Import correct host key or update client known_hosts"
  long_term: "Implement host key management and distribution procedures"
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

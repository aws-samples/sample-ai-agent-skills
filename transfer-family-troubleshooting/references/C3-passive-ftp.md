---
title: "C3 — Passive FTP Data Channel Issues"
description: "Diagnose FTP/FTPS passive mode data channel failures"
status: active
severity: MEDIUM
triggers:
  - "passive mode"
  - "PASV"
  - "EPSV"
  - "data channel"
  - "FTP transfer hangs"
  - "LIST command fails"
  - "FTP upload fails"
owner: devops-agent
objective: "Resolve FTP/FTPS passive mode data channel connection failures"
context: "Transfer Family uses passive mode for FTP and FTPS. In passive mode, the server tells the client which IP and port to connect to for the data channel. If the server is behind NAT/NLB, the PASV response may contain a private IP that external clients cannot reach. The data channel uses ports 8192-8200."
---

## Phase 1 — Triage

MUST:
- Verify FTP/FTPS is enabled: `aws transfer describe-server --server-id <server-id> --query 'Server.Protocols'`
- Check PassiveIp setting: `aws transfer describe-server --server-id <server-id> --query 'Server.ProtocolDetails.PassiveIp'`
- Check endpoint type and network topology: `aws transfer describe-server --server-id <server-id> --query 'Server.{EndpointType:EndpointType,EndpointDetails:EndpointDetails}'`
- Verify security groups allow ports 8192-8200 inbound: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Test FTP connection and capture PASV response: `curl -v ftp://<username>@<endpoint>/`

SHOULD:
- Check NACLs allow ports 8192-8200 in both directions
- Verify the PassiveIp matches the public-facing IP (EIP or NLB IP)
- Check if the client supports EPSV (Extended Passive) mode
- Test with a different FTP client to rule out client-specific issues

MAY:
- Capture network traffic to see the actual PASV response IP
- Check if the client is configured to use active mode (not supported by Transfer Family)
- Verify no intermediate proxy is rewriting FTP commands

## Phase 2 — Remediate

MUST:
- Set PassiveIp to the public-facing IP when the server is behind NAT or NLB: `aws transfer update-server --server-id <server-id> --protocol-details PassiveIp=<public-ip>`
- Open ports 8192-8200 in security groups and NACLs
- Ensure the client uses passive mode (Transfer Family does not support active mode)

SHOULD:
- Use EPSV mode if the client supports it (better NAT traversal)
- Document the PassiveIp configuration for operational reference
- Consider migrating to SFTP to avoid passive mode complexity

MAY:
- Set up monitoring for FTP data channel connection failures
- Implement automated PassiveIp updates if EIPs change

## Common Issues

- symptoms: "FTP login succeeds but LIST/GET/PUT commands hang"
  diagnosis: "PassiveIp not configured — PASV response contains private IP that client cannot reach."
  resolution: "Set PassiveIp to the public-facing IP address."

- symptoms: "FTP data transfer fails with 'connection refused' on data port"
  diagnosis: "Security group or NACL does not allow inbound on ports 8192-8200."
  resolution: "Add inbound rules for TCP ports 8192-8200 from client CIDR."

- symptoms: "FTP works from within VPC but not from external clients"
  diagnosis: "PassiveIp is set to the private IP or not set at all. External clients need the public IP."
  resolution: "Set PassiveIp to the Elastic IP or NLB public IP."

## Output Format

```yaml
root_cause: "passive_ftp — <specific_cause>"
evidence:
  - type: passive_ip
    content: "<PassiveIp configuration>"
  - type: pasv_response
    content: "<actual PASV response from server>"
  - type: security_groups
    content: "<port 8192-8200 rules>"
severity: MEDIUM
mitigation:
  immediate: "Configure PassiveIp and open data channel ports"
  long_term: "Consider migrating to SFTP to eliminate passive mode complexity"
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

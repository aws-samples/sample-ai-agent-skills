---
title: "C2 — Firewall and NAT Traversal Issues"
description: "Diagnose connectivity issues caused by firewalls, NAT gateways, and network address translation"
status: active
severity: MEDIUM
triggers:
  - "firewall blocking"
  - "NAT gateway"
  - "NAT traversal"
  - "corporate firewall"
  - "proxy"
  - "connection drops"
owner: devops-agent
objective: "Resolve connectivity issues caused by intermediate network devices between clients and Transfer Family servers"
context: "Firewalls, NAT gateways, and proxies can interfere with Transfer Family connections. SFTP (port 22) is often blocked by corporate firewalls. FTP/FTPS passive mode requires additional ports (8192-8200) that may be blocked. NAT gateways can cause issues with FTP passive mode IP addressing."
---

## Phase 1 — Triage

MUST:
- Identify the network path between client and server (direct, through NAT, through firewall, through proxy)
- Check if the client is behind a corporate firewall: test outbound connectivity on required ports
- For FTP/FTPS, verify both control (21) and data (8192-8200) ports are open through the firewall
- Check if NAT gateway is in the path and verify PassiveIp configuration: `aws transfer describe-server --server-id <server-id> --query 'Server.ProtocolDetails.PassiveIp'`
- Test from outside the firewall/NAT to confirm server-side connectivity works

SHOULD:
- Check firewall logs for blocked connections on ports 22, 21, 8192-8200
- Verify NAT gateway has sufficient Elastic IPs and is not hitting connection limits
- Check if the firewall performs deep packet inspection that interferes with SSH/TLS
- Verify the firewall allows the SFTP subsystem (some firewalls block SSH but allow SFTP or vice versa)

MAY:
- Check if a web proxy is intercepting connections (common in corporate environments)
- Verify MTU settings are not causing packet fragmentation issues
- Test with a packet capture to identify where the connection fails

## Phase 2 — Remediate

MUST:
- Open required ports in the firewall: 22 (SFTP), 21 + 8192-8200 (FTP/FTPS)
- Set PassiveIp in ProtocolDetails to the public-facing IP when FTP/FTPS is behind NAT
- Ensure NAT gateway or firewall does not modify SSH/TLS packets

SHOULD:
- Use SFTP (single port 22) instead of FTP/FTPS to simplify firewall rules
- Document firewall requirements for Transfer Family connectivity
- Work with network team to create specific firewall rules rather than broad exceptions

MAY:
- Consider using AWS Direct Connect or VPN for reliable private connectivity
- Implement a bastion host or jump box if direct firewall changes are not possible

## Common Issues

- symptoms: "SFTP control connection works but FTP file transfers fail"
  diagnosis: "Firewall allows port 21 but blocks passive data ports 8192-8200."
  resolution: "Open ports 8192-8200 in the firewall for FTP/FTPS data channels."

- symptoms: "FTP LIST command hangs after successful login"
  diagnosis: "PassiveIp not set — server sends private IP in PASV response, client cannot reach it through NAT."
  resolution: "Set PassiveIp to the public-facing IP in ProtocolDetails."

- symptoms: "Intermittent connection drops during large transfers"
  diagnosis: "Firewall or NAT gateway idle timeout closes the connection. Default TCP idle timeout may be too short."
  resolution: "Configure keep-alive on the client. Increase firewall/NAT idle timeout. Use SFTP which handles keep-alive better."

## Output Format

```yaml
root_cause: "firewall_nat — <specific_cause>"
evidence:
  - type: network_path
    content: "<identified network path and devices>"
  - type: firewall
    content: "<firewall rules and blocked ports>"
  - type: passive_ip
    content: "<PassiveIp configuration>"
severity: MEDIUM
mitigation:
  immediate: "Open required ports or configure PassiveIp"
  long_term: "Document network requirements and simplify with SFTP"
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

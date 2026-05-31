---
title: "C1 — Transfer Family Connection Failures"
description: "Diagnose why clients cannot establish connections to Transfer Family servers"
status: active
severity: HIGH
triggers:
  - "connection refused"
  - "connection timeout"
  - "cannot connect"
  - "connection reset"
  - "network unreachable"
  - "SFTP connection failed"
owner: devops-agent
objective: "Identify and resolve network-level connection failures to Transfer Family servers"
context: "Connection failures can occur at multiple layers: DNS resolution, network routing, security groups, NACLs, firewalls, or the Transfer Family server itself. The server must be in ONLINE state and the correct ports must be accessible from the client network."
---

## Phase 1 — Triage

MUST:
- Verify server is ONLINE: `aws transfer describe-server --server-id <server-id> --query 'Server.State'`
- Check endpoint type and hostname: `aws transfer describe-server --server-id <server-id> --query 'Server.{EndpointType:EndpointType,Endpoint:EndpointDetails}'`
- Test DNS resolution: `nslookup <server-id>.server.transfer.<region>.amazonaws.com`
- Test port connectivity: `nc -zv <endpoint> 22` (SFTP) or `nc -zv <endpoint> 21` (FTP/FTPS)
- For VPC endpoints, check security group inbound rules: `aws ec2 describe-security-groups --group-ids <sg-ids>`

SHOULD:
- Check NACLs on VPC endpoint subnets: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify route tables have correct routes: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Check if the client IP is in an allowed CIDR range in security groups
- Test from a different network to isolate client-side vs server-side issues

MAY:
- Check VPC Flow Logs for rejected packets: `aws logs filter-log-events --log-group-name <flow-log-group> --filter-pattern "REJECT"`
- Verify no AWS WAF or Network Firewall is blocking traffic
- Check if the server endpoint resolves to the expected IP addresses

## Phase 2 — Remediate

MUST:
- Start the server if it is OFFLINE: `aws transfer start-server --server-id <server-id>`
- Fix security group rules to allow inbound traffic on required ports from client CIDRs
- Fix NACL rules to allow both inbound and outbound traffic (including ephemeral ports 1024-65535)

SHOULD:
- Ensure DNS resolution returns the correct IP addresses
- Verify no intermediate firewalls or proxies are blocking the connection
- Test connectivity from the client's actual network, not just from within AWS

MAY:
- Enable VPC Flow Logs for ongoing network troubleshooting
- Implement connection monitoring with CloudWatch alarms

## Common Issues

- symptoms: "Connection timeout from external client"
  diagnosis: "Security group does not allow inbound from the client's public IP, or NACLs block the traffic."
  resolution: "Add the client's IP/CIDR to the security group inbound rules on the correct ports."

- symptoms: "Connection refused on port 22"
  diagnosis: "Server is OFFLINE, SFTP protocol is not enabled, or a firewall is sending RST."
  resolution: "Start the server, verify SFTP is in the Protocols list, check intermediate firewalls."

- symptoms: "DNS resolution fails for server endpoint"
  diagnosis: "VPC_ENDPOINT type is only resolvable from within the VPC. Or DNS propagation delay."
  resolution: "Use VPC endpoint type for public access. For VPC_ENDPOINT, connect from within the VPC."

## Output Format

```yaml
root_cause: "connection_failure — <specific_cause>"
evidence:
  - type: server_state
    content: "<server state>"
  - type: network
    content: "<security groups, NACLs, connectivity test results>"
  - type: dns
    content: "<DNS resolution results>"
severity: HIGH
mitigation:
  immediate: "Fix network configuration or start the server"
  long_term: "Implement network monitoring and connection health checks"
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

---
title: "D1 — Connection Failures"
description: "Diagnose Amazon Keyspaces connection failures"
status: active
severity: HIGH
triggers:
  - "connection refused"
  - "cannot connect"
  - "connection timeout"
  - "endpoint unreachable"
  - "cqlsh connection failed"
owner: devops-agent
objective: "Identify and resolve connection failures to Amazon Keyspaces"
context: "Keyspaces requires TLS connections on port 9142 to cassandra.<region>.amazonaws.com. Connection failures can be caused by incorrect endpoint, missing TLS configuration, network issues (security groups, NACLs, VPC endpoints), or authentication problems. VPC endpoint connections use interface endpoints."
---

## Phase 1 — Triage

MUST:
- Verify the endpoint and port: `nslookup cassandra.<region>.amazonaws.com` and confirm port 9142
- Check TLS certificate configuration in the client
- Verify security group allows outbound HTTPS/9142: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[*].IpPermissionsEgress'`
- Check if using VPC endpoint: `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.cassandra`

SHOULD:
- Test connectivity: `openssl s_client -connect cassandra.<region>.amazonaws.com:9142`
- Check NACLs for the subnet: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify DNS resolution from the client network
- Check route tables for internet or VPC endpoint routing

MAY:
- Check VPC endpoint security groups if using private connectivity
- Review client driver connection pool settings
- Check for regional service availability

## Phase 2 — Remediate

MUST:
- Configure TLS with Starfield certificate or Amazon root CA
- Use port 9142 (not 9042)
- Use correct regional endpoint: cassandra.<region>.amazonaws.com
- Ensure network path allows traffic on port 9142

SHOULD:
- Create VPC interface endpoint for private subnet access: `aws ec2 create-vpc-endpoint --vpc-id <vpc-id> --service-name com.amazonaws.<region>.cassandra --vpc-endpoint-type Interface --subnet-ids <subnet-ids> --security-group-ids <sg-ids>`
- Configure security groups on VPC endpoint to allow inbound 9142

MAY:
- Implement connection pooling and retry logic in the application
- Set up monitoring for connection failures

## Common Issues

- symptoms: "Connection refused on port 9042"
  diagnosis: "Using Cassandra default port instead of Keyspaces port."
  resolution: "Change port to 9142. Keyspaces does not listen on 9042."

- symptoms: "SSL handshake failure"
  diagnosis: "Missing or incorrect TLS certificate configuration."
  resolution: "Download Starfield certificate and configure in client SSL settings."

- symptoms: "Connection timeout from private subnet"
  diagnosis: "No route to Keyspaces endpoint. Missing VPC endpoint or NAT gateway."
  resolution: "Create VPC interface endpoint for com.amazonaws.<region>.cassandra or configure NAT gateway."

## Output Format

```yaml
root_cause: "connection_failure — <specific_cause>"
evidence:
  - type: endpoint_config
    content: "<endpoint, port, TLS settings>"
  - type: network_path
    content: "<security groups, route tables, VPC endpoints>"
  - type: tls_status
    content: "<TLS handshake result>"
severity: HIGH
mitigation:
  immediate: "Fix endpoint, port, TLS, or network configuration"
  long_term: "Implement VPC endpoints and connection monitoring"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never drop keyspaces or tables containing production data
- Never modify capacity mode on tables during peak traffic without load testing
- Never alter TTL settings without understanding data retention requirements
- Never change encryption settings on tables with active workloads
- Never disable point-in-time recovery on production tables

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore tables from point-in-time recovery if data is accidentally modified
- Revert capacity mode changes if throttling increases after modification
- Re-enable previous auto-scaling configurations if performance degrades

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

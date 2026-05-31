---
title: "C1 — Connection Failures"
description: "Diagnose DocumentDB connection failures and timeout issues"
status: active
severity: HIGH
triggers:
  - "connection refused"
  - "connection timeout"
  - "cannot connect"
  - "connection error"
  - "network timeout"
owner: devops-agent
objective: "Identify and resolve connection failures to DocumentDB clusters"
context: "DocumentDB connection failures are commonly caused by security group rules, TLS certificate issues, VPC configuration, DNS resolution, or connection limit exhaustion. DocumentDB is VPC-only and not publicly accessible."
---

## Phase 1 — Triage

MUST:
- Verify cluster is available: `aws docdb describe-db-clusters --db-cluster-identifier <cluster-id>`
- Check security group allows port 27017: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Verify client is in the same VPC or has VPC peering/TGW connectivity
- Check TLS certificate is specified in connection string

SHOULD:
- Test DNS resolution of cluster endpoint from the client
- Check DatabaseConnections metric against limits: `aws cloudwatch get-metric-statistics --namespace AWS/DocDB --metric-name DatabaseConnections --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Verify subnet routing between client and DocumentDB subnets
- Check NACLs for port 27017 blocking

MAY:
- Test connectivity with: `openssl s_client -connect <endpoint>:27017`
- Check VPC flow logs for dropped packets
- Verify no proxy or firewall is interfering

## Phase 2 — Remediate

MUST:
- Add security group rule allowing inbound TCP 27017 from client
- Ensure TLS certificate (rds-combined-ca-bundle.pem) is available on the client
- Verify VPC connectivity between client and DocumentDB

SHOULD:
- Use the cluster endpoint for writer connections and reader endpoint for reads
- Implement connection pooling to avoid connection exhaustion
- Monitor DatabaseConnections metric

MAY:
- Set up VPC endpoints for cross-VPC access
- Configure connection timeout and retry parameters in the driver

## Common Issues

- symptoms: "Connection timeout from EC2 instance"
  diagnosis: "Security group does not allow inbound TCP 27017 from the client security group."
  resolution: "Add inbound rule for TCP 27017 from the client's security group or CIDR."

- symptoms: "SSL handshake failure"
  diagnosis: "TLS certificate is missing, expired, or incorrect in the connection string."
  resolution: "Download the latest rds-combined-ca-bundle.pem and specify it with --tlsCAFile."

- symptoms: "Connection refused — too many connections"
  diagnosis: "Instance connection limit reached. Each instance class has a maximum connection count."
  resolution: "Implement connection pooling. Scale up instance class for higher connection limits."

## Output Format

```yaml
root_cause: "connection_failure — <specific_cause>"
evidence:
  - type: cluster
    content: "<cluster status and endpoints>"
  - type: security_group
    content: "<security group rules>"
  - type: connectivity
    content: "<network connectivity test results>"
severity: HIGH
mitigation:
  immediate: "Fix security group, TLS, or network configuration"
  long_term: "Implement connection pooling and monitoring"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Connection troubleshooting may expose credentials and network architecture |
| Blast Radius | HIGH — Connection issues affect all applications using the cluster |
| Rollback Complexity | LOW — Security group and network changes can be reverted |
| Compliance Scope | HIGH — Database connectivity is critical for application availability |

## Escalation Conditions

- Escalate to network team if VPC routing issues prevent connectivity
- Escalate to security if security group changes are needed
- Escalate to operations if connection exhaustion affects application availability

## Data Sensitivity

- **Connection strings** may contain credentials — never log in plain text
- **Security group rules** reveal network access patterns
- **TLS certificates** are required for secure connections

## PROHIBITED Actions

- Do NOT open security groups to 0.0.0.0/0 to fix connectivity issues
- Do NOT log connection strings containing credentials
- Do NOT increase connection limits without implementing connection pooling
- Do NOT bypass TLS to troubleshoot connection issues in production

## Phase 3 — Rollback

- Revert security group changes if they were modified
- Restore previous TLS configuration if it was changed
- Revert NACL changes if they were modified
- Verify all applications can connect after rollback

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
  - "NEVER suggest disabling TLS"
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"

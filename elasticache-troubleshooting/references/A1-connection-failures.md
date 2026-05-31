---
title: "A1 — ElastiCache Connection Failures"
description: "Diagnose why clients cannot connect to ElastiCache Redis or Memcached nodes"
status: active
severity: CRITICAL
triggers:
  - "connection refused"
  - "cannot connect"
  - "connection failed"
  - "ECONNREFUSED"
  - "no route to host"
owner: devops-agent
objective: "Identify and resolve connection failures between application clients and ElastiCache nodes"
context: "ElastiCache clusters run inside a VPC and are not publicly accessible. Connection failures typically stem from security group misconfiguration, subnet routing issues, incorrect endpoints, or node unavailability. Redis uses port 6379 (or 6380 for TLS), Memcached uses port 11211."
---

## Phase 1 — Triage

MUST:
- Check cluster status: `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --show-cache-node-info`
- Check replication group status (Redis): `aws elasticache describe-replication-groups --replication-group-id <repl-group-id>`
- Verify security group allows inbound on correct port: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[*].IpPermissions'`
- Check subnet group: `aws elasticache describe-cache-subnet-groups --cache-subnet-group-name <subnet-group>`
- Test connectivity from client: `redis-cli -h <endpoint> -p 6379 PING` (Redis) or `telnet <endpoint> 11211` (Memcached)
- Check recent events: `aws elasticache describe-events --source-identifier <cluster-id> --source-type cache-cluster --duration 60`

SHOULD:
- Verify the client is in the same VPC or has VPC peering/Transit Gateway connectivity
- Check if TLS is required: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].TransitEncryptionEnabled'`
- Check current connections: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name CurrConnections --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Verify NACLs on the subnet allow traffic on the ElastiCache port

MAY:
- Check VPC flow logs for rejected traffic
- Verify DNS resolution of the endpoint: `nslookup <endpoint>`
- Check if the node is undergoing maintenance: `aws elasticache describe-service-updates`

## Phase 2 — Remediate

MUST:
- Fix security group rules to allow inbound from client security group on port 6379 (Redis) or 11211 (Memcached)
- Ensure client is in a subnet that can route to the ElastiCache subnet
- Use the correct endpoint (primary endpoint for writes, reader endpoint for reads, configuration endpoint for Memcached)
- If TLS is enabled, ensure client uses TLS connection (redis-cli --tls or application TLS config)

SHOULD:
- Implement connection retry logic with exponential backoff in the application
- Use connection pooling to avoid connection storms
- Monitor CurrConnections metric and set CloudWatch alarms

MAY:
- Set up a bastion host for debugging connectivity from outside the VPC
- Enable VPC flow logs for ongoing network troubleshooting

## Common Issues

- symptoms: "Connection refused on port 6379"
  diagnosis: "Security group does not allow inbound on port 6379 from the client's security group or CIDR."
  resolution: "Add inbound rule: Type=Custom TCP, Port=6379, Source=<client-sg-id>."

- symptoms: "Connection timeout (no response)"
  diagnosis: "Client is in a different VPC or subnet with no route to the ElastiCache subnet."
  resolution: "Ensure VPC peering, Transit Gateway, or same-VPC connectivity. Check route tables."

- symptoms: "Connection works with redis-cli but not with TLS"
  diagnosis: "Cluster has in-transit encryption enabled but client is not using TLS."
  resolution: "Use redis-cli --tls or configure application Redis client with TLS enabled."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix security group rules | GREEN | Network configuration; no data impact |
| Ensure correct endpoint usage | GREEN | Application-level configuration change |
| Enable TLS connection in client | GREEN | Application-level security improvement |
| Set up bastion host | YELLOW | New infrastructure; requires security review |

## Escalation Conditions

- Connection failure affecting production application availability
- Security group changes required on a shared security group used by multiple services
- TLS configuration mismatch preventing all client connections
- VPC peering or Transit Gateway changes needed for cross-VPC connectivity
- Connection failure persists after security group and endpoint corrections

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-cache-clusters` | MEDIUM | Exposes cluster configuration and endpoints |
| `describe-security-groups` | MEDIUM | Exposes network security rules |
| `describe-cache-subnet-groups` | LOW | Subnet configuration only |
| `redis-cli PING` | LOW | Connectivity test only; no data access |
| `describe-events` | LOW | Operational events only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to fix connection issues
- NEVER suggest disabling AUTH to troubleshoot connection failures
- NEVER suggest disabling encryption in transit to simplify connectivity
- NEVER suggest reducing node count during peak traffic to resolve connection issues
- NEVER open security groups to 0.0.0.0/0 for ElastiCache access

## Phase 3 — Rollback

If connection remediation changes cause issues:
1. Revert security group rules to previous configuration: `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol tcp --port 6379 --source-group <new-source-sg>`
2. If endpoint was changed in application, revert to previous endpoint configuration
3. If TLS was enabled in client but causes handshake failures, revert to non-TLS connection (if cluster supports it)
4. If bastion host was set up, terminate if no longer needed to reduce attack surface
5. Verify application connectivity is restored after rollback

## Output Format

```yaml
root_cause: "connection_failure — <specific_cause>"
evidence:
  - type: cluster_status
    content: "<cluster status and node info>"
  - type: security_group
    content: "<inbound rules>"
  - type: connectivity_test
    content: "<PING result or telnet output>"
severity: CRITICAL
mitigation:
  immediate: "Fix security group or network routing to restore connectivity"
  long_term: "Implement connection monitoring, retry logic, and connection pooling"
```

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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest disabling AUTH"
  - "NEVER suggest public subnet placement"

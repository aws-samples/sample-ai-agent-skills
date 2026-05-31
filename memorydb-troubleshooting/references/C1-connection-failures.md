---
title: "C1 — Connection Failures"
description: "Diagnose MemoryDB connection failures and timeout issues"
status: active
severity: HIGH
triggers:
  - "connection refused"
  - "connection timeout"
  - "cannot connect"
  - "TLS error"
  - "MOVED error"
owner: devops-agent
objective: "Identify and resolve connection failures to MemoryDB clusters"
context: "MemoryDB connections require TLS, cluster-aware clients, and proper ACL authentication. Connection failures stem from security group rules, TLS configuration, non-cluster-aware clients, ACL issues, or VPC connectivity."
---

## Phase 1 — Triage

MUST:
- Verify cluster is available: `aws memorydb describe-clusters --cluster-name <cluster-name>`
- Check security group allows port 6379: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Verify client uses TLS (required for MemoryDB)
- Verify client uses a Redis Cluster-aware driver

SHOULD:
- Test connectivity: `redis-cli --tls -h <cluster-endpoint> -p 6379 PING`
- Check connection count: `aws cloudwatch get-metric-statistics --namespace AWS/MemoryDB --metric-name CurrConnections --dimensions Name=ClusterName,Value=<cluster-name> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Verify ACL authentication credentials
- Check subnet routing between client and MemoryDB

MAY:
- Check VPC flow logs for dropped packets
- Verify NACLs allow port 6379
- Test from different client locations

## Phase 2 — Remediate

MUST:
- Add security group rule for inbound TCP 6379
- Configure client with TLS support
- Use Redis Cluster-aware client library
- Provide correct ACL username and password

SHOULD:
- Use the cluster configuration endpoint for automatic node discovery
- Implement connection pooling
- Configure connection retry logic

MAY:
- Set up VPC endpoints for cross-VPC access
- Monitor connection metrics

## Common Issues

- symptoms: "MOVED error from Redis client"
  diagnosis: "Client is not cluster-aware and cannot follow hash slot redirections."
  resolution: "Use a Redis Cluster-aware client (JedisCluster, redis-py-cluster, ioredis cluster mode)."

- symptoms: "TLS handshake failure"
  diagnosis: "Client does not have TLS enabled or is using an incompatible TLS version."
  resolution: "Enable TLS in the client. Use TLS 1.2+. Use redis-cli with --tls flag."

- symptoms: "NOAUTH error"
  diagnosis: "Client did not provide ACL credentials or credentials are incorrect."
  resolution: "Provide username and password in the connection. Verify ACL user exists and is active."

## Output Format

```yaml
root_cause: "connection_failure — <specific_cause>"
evidence:
  - type: cluster
    content: "<cluster status and endpoint>"
  - type: security_group
    content: "<security group rules>"
  - type: connectivity
    content: "<connection test results>"
severity: HIGH
mitigation:
  immediate: "Fix security group, TLS, or client configuration"
  long_term: "Standardize client configuration and implement monitoring"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete a MemoryDB cluster without verifying snapshot backups and data durability requirements
- **NEVER** modify ACL rules on production clusters without testing authentication changes in a staging environment
- **NEVER** force a failover on a production cluster during peak traffic without customer approval
- **NEVER** reduce the number of shards or replicas on a cluster serving production workloads without capacity analysis
- **NEVER** disable encryption in-transit or at-rest on clusters handling sensitive data

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
  - "NEVER suggest removing ACL authentication"
  - "NEVER suggest public subnet placement"

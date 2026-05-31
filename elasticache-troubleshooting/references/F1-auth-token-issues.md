---
title: "F1 — ElastiCache AUTH Token Issues"
description: "Diagnose Redis AUTH token authentication failures and token rotation problems"
status: active
severity: HIGH
triggers:
  - "AUTH token"
  - "authentication failed"
  - "NOAUTH"
  - "ERR invalid password"
  - "token rotation"
  - "Redis AUTH"
owner: devops-agent
objective: "Resolve AUTH token authentication failures and ensure smooth token rotation"
context: "Redis AUTH provides password-based authentication. ElastiCache supports AUTH tokens (single password, pre-6.x) and Redis ACLs (multiple users with per-command permissions, 6.x+). IAM authentication is available in ElastiCache 7.0+. AUTH token rotation uses modify-replication-group with ROTATE (allows both old and new tokens) or SET (immediate replacement). AUTH can only be enabled at cluster creation."
---

## Phase 1 — Triage

MUST:
- Check if AUTH is enabled: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].{AuthEnabled:AuthTokenEnabled,AuthLastModified:AuthTokenLastModifiedDate}'`
- Test AUTH with current token: `redis-cli -h <endpoint> -p 6379 AUTH <token> PING`
- Check Redis version (ACL support requires 6.x+): `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --query 'CacheClusters[*].EngineVersion'`
- Check recent events for AUTH changes: `aws elasticache describe-events --source-type replication-group --duration 1440`
- Check if TLS is required alongside AUTH: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].TransitEncryptionEnabled'`

SHOULD:
- Check if AUTH token rotation is in progress (ROTATE strategy allows dual tokens)
- Verify the application is using the correct token
- Check if ACLs are configured (Redis 6.x+): `redis-cli -h <endpoint> -p 6379 ACL LIST`
- Check CloudTrail for modify-replication-group calls with auth-token changes

MAY:
- Check if IAM authentication is available (Redis 7.0+)
- Review application connection string configuration
- Check if the AUTH token meets complexity requirements (16-128 printable characters)

## Phase 2 — Remediate

MUST:
- For token rotation, use ROTATE strategy for zero-downtime: `aws elasticache modify-replication-group --replication-group-id <repl-group-id> --auth-token <new-token> --auth-token-update-strategy ROTATE --apply-immediately`
- Update all application clients with the new token
- After all clients are updated, finalize rotation with SET: `aws elasticache modify-replication-group --replication-group-id <repl-group-id> --auth-token <new-token> --auth-token-update-strategy SET --apply-immediately`

SHOULD:
- Store AUTH tokens in AWS Secrets Manager with automatic rotation
- Use Redis ACLs (6.x+) for granular per-user, per-command permissions
- Consider migrating to IAM authentication (7.0+) to eliminate password management

MAY:
- Implement token rotation automation using Lambda and Secrets Manager
- Set up monitoring for AUTH failure events
- Use separate ACL users for different application components

## Common Issues

- symptoms: "NOAUTH Authentication required"
  diagnosis: "AUTH is enabled but the client is not sending the AUTH command."
  resolution: "Configure the Redis client to send AUTH with the correct token on connection."

- symptoms: "ERR invalid password after token rotation"
  diagnosis: "Application is using the old token after SET strategy was applied."
  resolution: "Use ROTATE strategy first, update all clients, then use SET to finalize."

- symptoms: "Cannot enable AUTH on existing cluster"
  diagnosis: "AUTH can only be enabled at cluster creation time."
  resolution: "Create a new cluster with AUTH enabled and migrate data."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Rotate AUTH token with ROTATE strategy | YELLOW | Allows dual tokens; requires updating all clients before SET |
| Finalize rotation with SET strategy | RED | Immediately invalidates old token; clients using old token will fail |
| Store token in Secrets Manager | GREEN | Security improvement; no operational impact |
| Migrate to Redis ACLs | YELLOW | Requires Redis 6.x+; application changes needed |
| Migrate to IAM authentication | YELLOW | Requires Redis 7.0+; eliminates password management |

## Escalation Conditions

- AUTH failure preventing all client connections in production
- Token rotation in progress but some clients not yet updated (dual-token window)
- Cannot enable AUTH on existing cluster (requires new cluster creation)
- AUTH token compromised requiring emergency rotation
- ACL misconfiguration locking out administrative access

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-replication-groups` (AuthTokenEnabled) | LOW | Boolean flag only |
| `AUTH <token> PING` | HIGH | Transmits the AUTH token; use only over TLS |
| `ACL LIST` | HIGH | Exposes user permissions and may reveal password hashes |
| `describe-events` (AUTH changes) | LOW | Operational events only |
| `describe-cache-clusters` (EngineVersion) | LOW | Version information only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to fix AUTH issues
- NEVER suggest disabling AUTH to resolve authentication failures (security regression)
- NEVER suggest disabling encryption in transit when AUTH is enabled (tokens transmitted in cleartext)
- NEVER suggest reducing node count during peak traffic
- NEVER log or expose AUTH tokens in plaintext in application logs or monitoring

## Phase 3 — Rollback

If AUTH token changes cause issues:
1. If ROTATE strategy was used and new token causes issues, clients can still use the old token until SET is applied
2. If SET strategy was applied prematurely, rotate again with the working token: `aws elasticache modify-replication-group --replication-group-id <repl-group-id> --auth-token <working-token> --auth-token-update-strategy ROTATE --apply-immediately`
3. If ACL changes locked out users, use the default user to restore access
4. If IAM authentication was enabled and causes issues, revert to token-based AUTH
5. Update all application clients with the correct token after rollback

## Output Format

```yaml
root_cause: "auth_token — <specific_cause>"
evidence:
  - type: auth_config
    content: "<AUTH enabled status and last modified>"
  - type: auth_test
    content: "<AUTH test result>"
  - type: redis_version
    content: "<engine version>"
severity: HIGH
mitigation:
  immediate: "Fix AUTH token or update application configuration"
  long_term: "Implement automated token rotation with Secrets Manager"
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

---
title: "I2 — IAM Database Authentication"
description: "Diagnose Aurora IAM database authentication issues"
status: active
severity: MEDIUM
triggers:
  - "IAM authentication"
  - "IAM database auth"
  - "rds-db:connect"
  - "auth token"
  - "generate-db-auth-token"
owner: devops-agent
objective: "Identify and resolve Aurora IAM database authentication issues"
context: "Aurora supports IAM database authentication for both MySQL and PostgreSQL. Instead of passwords, users authenticate with temporary tokens generated via AWS STS. Requires enabling IAM auth on the cluster, creating a database user mapped to IAM, and granting rds-db:connect permission. Tokens are valid for 15 minutes. Connection must use SSL."
---

## Phase 1 — Triage

MUST:
- Check if IAM auth is enabled on the cluster:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].IAMDatabaseAuthenticationEnabled'
  ```
- Verify IAM policy has rds-db:connect permission:
  ```
  aws iam simulate-principal-policy --policy-source-arn <iam-user-or-role-arn> \
    --action-names rds-db:connect \
    --resource-arns "arn:aws:rds-db:<region>:<account-id>:dbuser:<resource-id>/<db-username>"
  ```
- Check database user configuration:
  - Aurora MySQL:
    ```sql
    SELECT user, host, plugin FROM mysql.user WHERE user = '<iam-db-user>';
    -- Plugin should be 'AWSAuthenticationPlugin' (MySQL 5.7) or 'aws_iam' (MySQL 8.0)
    ```
  - Aurora PostgreSQL:
    ```sql
    SELECT usename, valuntil FROM pg_user WHERE usename = '<iam-db-user>';
    -- Check if user has rds_iam role
    SELECT r.rolname FROM pg_roles r
    JOIN pg_auth_members m ON r.oid = m.roleid
    JOIN pg_roles u ON u.oid = m.member
    WHERE u.rolname = '<iam-db-user>' AND r.rolname = 'rds_iam';
    ```
- Generate and test auth token:
  ```
  aws rds generate-db-auth-token --hostname <cluster-endpoint> --port <port> --username <db-username>
  ```

SHOULD:
- Verify SSL is being used (IAM auth requires SSL)
- Check that the resource ID in the IAM policy matches the cluster resource ID:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].DbClusterResourceId'
  ```
- Verify the token is being used within 15 minutes of generation

MAY:
- Check CloudTrail for rds-db:connect authorization events
- Review if RDS Proxy with IAM auth is a better approach

## Phase 2 — Remediate

MUST:
- Enable IAM auth if not enabled:
  ```
  aws rds modify-db-cluster --db-cluster-identifier <cluster-id> --enable-iam-database-authentication
  ```
- Create database user for IAM auth:
  - Aurora MySQL:
    ```sql
    CREATE USER '<iam-db-user>'@'%' IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS';
    GRANT SELECT, INSERT, UPDATE, DELETE ON <database>.* TO '<iam-db-user>'@'%';
    ```
  - Aurora PostgreSQL:
    ```sql
    CREATE USER <iam-db-user>;
    GRANT rds_iam TO <iam-db-user>;
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO <iam-db-user>;
    ```
- Grant IAM policy with rds-db:connect on the correct resource ARN

SHOULD:
- Use the cluster resource ID (not cluster identifier) in the IAM policy resource ARN
- Ensure client connects with SSL (required for IAM auth)
- Use RDS Proxy for IAM auth to simplify token management

MAY:
- Implement token caching in the application (tokens valid for 15 minutes)
- Use IAM roles for EC2/Lambda instead of IAM users for better security

## Common Issues

- symptoms: "Access denied when using IAM auth token"
  diagnosis: "IAM policy resource ARN incorrect, database user not configured, or SSL not used."
  resolution: "Verify resource ARN uses DbClusterResourceId. Check database user plugin. Use SSL."

- symptoms: "Token expired"
  diagnosis: "Auth token used more than 15 minutes after generation."
  resolution: "Generate a fresh token before each connection or implement token caching with refresh."

- symptoms: "IAM auth works but connection is slow"
  diagnosis: "Token generation adds latency to each connection."
  resolution: "Use RDS Proxy with IAM auth. Cache tokens. Use connection pooling."

## Safety Ratings
- GREEN: describe-db-clusters (IAM auth status), iam simulate-principal-policy, generate-db-auth-token, SELECT from mysql.user/pg_roles — read-only inspection
- YELLOW: modify-db-cluster (enable IAM auth), CREATE USER, GRANT rds_iam — recoverable changes
- RED: delete-db-cluster, DROP USER — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "IAM authentication failure affecting application connectivity"
- "Fix requires modifying IAM policies or database users"
- "Fix involves changing cluster authentication settings"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: IAM policies, role ARNs, database user configuration (core authentication infrastructure)
- HIGH: auth tokens (temporary but contain authentication material)
- HIGH: database credentials, connection strings, DbClusterResourceId

## Prohibited Actions
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest storing IAM auth tokens in application code or logs"
- "NEVER suggest granting overly broad IAM permissions (use least privilege)"
- "NEVER suggest using IAM auth without SSL — SSL is required"

## Phase 3 — Rollback
- "Revert IAM policy changes if they break authentication"
- "Revert database user changes if IAM auth configuration is incorrect"
- "If IAM auth causes widespread connection failures, temporarily revert to password authentication"
- "Restore from snapshot if authentication configuration change causes issues"

## Output Format

```yaml
root_cause: "iam_auth_issue — <specific_cause>"
evidence:
  - type: iam_config
    content: "<IAM auth enabled status, IAM policy>"
  - type: db_user
    content: "<database user configuration>"
  - type: token_test
    content: "<auth token generation result>"
severity: MEDIUM
mitigation:
  immediate: "Fix IAM policy or database user configuration"
  long_term: "Implement RDS Proxy with IAM auth for simplified management"
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
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling encryption"
  - "NEVER force failover without understanding impact"

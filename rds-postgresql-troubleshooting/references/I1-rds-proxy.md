---
title: "I1 — RDS Proxy Issues"
description: "Diagnose RDS Proxy connection pooling and authentication for PostgreSQL"
status: active
severity: HIGH
triggers:
  - "RDS Proxy"
  - "proxy"
  - "connection pooling"
---

## Phase 1 — Triage

MUST:
- Check proxy status: `aws rds describe-db-proxies --db-proxy-name <proxy-name>`
- Check target health: `aws rds describe-db-proxy-targets --db-proxy-name <proxy-name>`
- Check CloudWatch proxy metrics: `ClientConnections`, `DatabaseConnections`, `QueryRequests`
- Verify Secrets Manager secret: `aws secretsmanager get-secret-value --secret-id <secret-arn>`

SHOULD:
- Check IAM role permissions for proxy
- Verify proxy and RDS instance are in same VPC
- Check security groups allow proxy→RDS on port 5432
- Check for connection pinning issues

## Phase 2 — Remediate

- Auth failures: verify Secrets Manager secret credentials
- Connection limit: adjust `MaxConnectionsPercent`
- Pinning: PostgreSQL features like prepared statements, SET commands, advisory locks cause pinning — minimize usage for better pooling
- Target unavailable: check RDS instance health
- IAM auth: ensure proxy IAM role has `secretsmanager:GetSecretValue` and `kms:Decrypt`

## Safety Ratings
- GREEN: describe-db-proxies, describe-db-proxy-targets, describe-db-proxy-endpoints, CloudWatch proxy metrics — read-only inspection
- YELLOW: modify-db-proxy, modify-db-proxy-endpoint — recoverable configuration changes
- RED: delete-db-proxy, delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Proxy authentication failures affecting application connectivity"
- "Fix requires modifying Secrets Manager credentials"
- "Fix involves modifying IAM roles or policies"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: Secrets Manager secret ARNs and credentials (contain database passwords)
- HIGH: IAM role policies, proxy authentication configuration
- MEDIUM: proxy connection metrics, pinning statistics

## Prohibited Actions
- "NEVER suggest deleting an RDS Proxy to fix issues — reconfigure instead"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"
- "NEVER suggest storing database credentials in application code instead of Secrets Manager"

## Phase 3 — Rollback
- "Revert Secrets Manager secret to previous version if credential update fails"
- "Revert IAM policy changes if they break proxy authentication"
- "Revert proxy configuration changes if they affect connection routing"

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
  - "NEVER suggest making databases publicly accessible"
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting automated backups"

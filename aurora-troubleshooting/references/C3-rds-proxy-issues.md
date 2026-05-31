---
title: "C3 — RDS Proxy Issues"
description: "Diagnose RDS Proxy configuration and connectivity issues with Aurora"
status: active
severity: HIGH
triggers:
  - "RDS Proxy"
  - "proxy connection"
  - "proxy timeout"
  - "proxy authentication"
  - "proxy failover"
  - "connection pinning"
owner: devops-agent
objective: "Identify and resolve RDS Proxy issues when used with Aurora clusters"
context: "RDS Proxy provides connection pooling, failover routing, and IAM authentication for Aurora. Common issues include Secrets Manager configuration, IAM role permissions, connection pinning (which reduces pooling effectiveness), and proxy endpoint configuration."
---

## Phase 1 — Triage

MUST:
- Check proxy status:
  ```
  aws rds describe-db-proxies --db-proxy-name <proxy-name>
  ```
- Check proxy target group and health:
  ```
  aws rds describe-db-proxy-target-groups --db-proxy-name <proxy-name>
  aws rds describe-db-proxy-targets --db-proxy-name <proxy-name>
  ```
- Check proxy endpoints:
  ```
  aws rds describe-db-proxy-endpoints --db-proxy-name <proxy-name>
  ```
- Check CloudWatch metrics for the proxy:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections \
    --dimensions Name=ProxyName,Value=<proxy-name> ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ClientConnections ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name QueryRequests ...
  ```

SHOULD:
- Verify Secrets Manager secret exists and contains correct credentials:
  ```
  aws secretsmanager describe-secret --secret-id <secret-arn>
  ```
- Check IAM role permissions for the proxy:
  ```
  aws iam get-role --role-name <proxy-role>
  aws iam list-attached-role-policies --role-name <proxy-role>
  ```
- Check proxy logs in CloudWatch Logs (if logging enabled):
  ```
  aws logs filter-log-events --log-group-name /aws/rds/proxy/<proxy-name>
  ```
- Check for connection pinning issues:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnectionsCurrentlySessionPinned \
    --dimensions Name=ProxyName,Value=<proxy-name> ...
  ```

MAY:
- Check VPC security groups between proxy and Aurora cluster
- Verify proxy is in the same VPC as the Aurora cluster

## Phase 2 — Remediate

MUST:
- For authentication failures: verify Secrets Manager secret has correct username/password matching an Aurora database user
- For IAM role issues: ensure the proxy role has `secretsmanager:GetSecretValue` and `kms:Decrypt` permissions
- For target health issues: verify security group allows proxy to connect to Aurora on the database port

SHOULD:
- Reduce connection pinning by avoiding session-level variables and prepared statements where possible
- Configure appropriate idle client timeout and connection borrow timeout
- Use separate proxy endpoints for read and write workloads

MAY:
- Enable enhanced logging for debugging connection issues
- Implement proxy endpoint for reader routing

## Common Issues

- symptoms: "Proxy target state is UNAVAILABLE"
  diagnosis: "Proxy cannot connect to Aurora cluster. Security group or credential issue."
  resolution: "Check security group rules. Verify Secrets Manager credentials match Aurora user."

- symptoms: "High connection pinning percentage"
  diagnosis: "Session state (variables, prepared statements, temp tables) prevents connection reuse."
  resolution: "Minimize session state. Avoid SET statements. Use connection-level settings."

- symptoms: "Proxy authentication failure with IAM"
  diagnosis: "IAM policy missing rds-db:connect permission or wrong resource ARN."
  resolution: "Grant rds-db:connect on the proxy resource ARN (not the cluster ARN)."

## Safety Ratings
- GREEN: describe-db-proxies, describe-db-proxy-targets, describe-db-proxy-endpoints, CloudWatch proxy metrics, describe-secret — read-only inspection
- YELLOW: modify-db-proxy, modify-db-proxy-endpoint, update Secrets Manager secret — recoverable configuration changes
- RED: delete-db-proxy, force-failover — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires modifying proxy configuration affecting active connections"
- "Fix requires updating Secrets Manager credentials"
- "Fix involves modifying IAM roles or policies"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: Secrets Manager secret ARNs and credentials (contain database passwords)
- HIGH: IAM role policies, proxy authentication configuration
- MEDIUM: proxy connection metrics, pinning statistics, CloudWatch Logs entries

## Prohibited Actions
- "NEVER suggest deleting an RDS Proxy to fix issues — reconfigure instead"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest storing database credentials in application code instead of Secrets Manager"

## Phase 3 — Rollback
- "Restore from snapshot if proxy configuration change causes issues"
- "Revert Secrets Manager secret to previous version if credential update fails"
- "Revert IAM policy changes if they break proxy authentication"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "rds_proxy_issue — <specific_cause>"
evidence:
  - type: proxy_status
    content: "<proxy and target health>"
  - type: cloudwatch
    content: "<proxy metrics>"
  - type: logs
    content: "<proxy log entries>"
severity: HIGH
mitigation:
  immediate: "Fix proxy configuration or connectivity issue"
  long_term: "Implement proxy monitoring and optimize for minimal pinning"
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

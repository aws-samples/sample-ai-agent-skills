---
title: "H2 — MWAA Airflow Connection Issues"
description: "Diagnose Airflow connection configuration and connectivity problems in MWAA"
status: active
severity: MEDIUM
triggers:
  - "connection failed"
  - "connection not found"
  - "connection error"
  - "database connection"
  - "external service"
owner: devops-agent
objective: "Identify and resolve Airflow connection configuration issues in MWAA"
context: "Airflow connections define how DAGs connect to external systems (databases, APIs, AWS services). Connections can be stored in the Airflow metastore, Secrets Manager, or Parameter Store. Connection failures stem from incorrect credentials, network issues, or missing configuration."
---

## Phase 1 — Triage

MUST:
- List connections via CLI token: use `aws mwaa create-cli-token --name <env-name>` then `connections list`
- Check task logs for connection errors: `aws logs filter-log-events --log-group-name airflow-<env-name>-Task --filter-pattern "connection" --start-time <epoch-ms>`
- Verify connection exists: use CLI token with `connections get <conn_id>`
- Check if secrets backend is configured: `aws mwaa get-environment --name <env-name> --query 'Environment.AirflowConfigurationOptions'`

SHOULD:
- Test connection from Airflow UI if accessible
- Verify network connectivity to the target system (VPC, security groups)
- Check if connection credentials are expired or rotated

MAY:
- Check if the connection type requires a specific provider package in requirements.txt
- Verify connection extra parameters are valid JSON

## Phase 2 — Remediate

MUST:
- Create or update the connection with correct credentials
- Ensure network path exists between MWAA VPC and target system
- Install required provider packages in requirements.txt for the connection type

SHOULD:
- Use Secrets Manager for connection storage instead of metastore
- Test connections before using them in production DAGs
- Document connection requirements for each DAG

MAY:
- Implement connection health checks in DAGs
- Use connection pooling for database connections

## Common Issues

- symptoms: "Connection not found error in task logs"
  diagnosis: "Connection ID doesn't exist in metastore or secrets backend."
  resolution: "Create the connection via CLI or Secrets Manager with the correct conn_id."

- symptoms: "Connection exists but authentication fails"
  diagnosis: "Credentials are incorrect, expired, or the connection URI format is wrong."
  resolution: "Update connection credentials. Use JSON format for complex connection parameters."

- symptoms: "Connection timeout to external database"
  diagnosis: "MWAA VPC cannot reach the target database (security groups, routing)."
  resolution: "Verify security groups allow outbound traffic to the database port and host."

## Output Format

```yaml
root_cause: "connection_issue — <specific_cause>"
evidence:
  - type: connection_config
    content: "<connection details (redacted credentials)>"
  - type: task_logs
    content: "<connection error messages>"
  - type: network_config
    content: "<VPC/security group details>"
severity: MEDIUM
mitigation:
  immediate: "Fix connection configuration or network path"
  long_term: "Use Secrets Manager for all connections with rotation"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — connections contain database credentials and API keys
- Service Disruption Risk: MEDIUM — broken connections cause task failures
- Compliance Impact: HIGH — connection security affects data access compliance

## Escalation Conditions
- Connection credentials exposed in logs or UI
- Multiple connections fail simultaneously indicating infrastructure issue
- Connection to critical data source lost
- Unauthorized connection modifications detected
- Connection encryption or TLS configuration issues

## Data Sensitivity
- Database connection strings — RESTRICTED
- API keys and authentication tokens — RESTRICTED
- Connection passwords — RESTRICTED
- Host addresses and ports — CONFIDENTIAL
- Connection metadata — INTERNAL
- Never expose connection credentials in any output

## Prohibited Actions
- Do NOT store connection credentials in Airflow Variables
- Do NOT create connections with plaintext passwords when secrets backend is available
- Do NOT share connection credentials across environments
- Do NOT log connection details including passwords
- Do NOT grant Airflow UI connection edit access to unauthorized users
- Do NOT use unencrypted connections to sensitive data sources

## Phase 3 — Rollback
- Restore connection configurations from secrets backend
- Rotate credentials for any connections that may have been exposed
- Revert connection parameter changes to last known working values
- Verify task connectivity after connection changes
- Audit connection access logs for unauthorized modifications
- Document connection changes and update credential rotation schedule

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
  - "NEVER suggest disabling webserver authentication"
  - "NEVER suggest public network access mode for production"
  - "NEVER suggest overly broad Airflow execution role"

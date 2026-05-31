# F1 — Athena Integration Issues

## Symptoms
- Athena queries fail on discovery data
- Tables not found in Athena
- Query results empty

## Triage Commands

```bash
aws discovery describe-continuous-exports --max-results 10
aws athena list-databases --catalog-name AwsDataCatalog
aws athena list-table-metadata --catalog-name AwsDataCatalog --database-name <discovery-db>
```

## Common Causes

1. **Continuous export not active** — Must be running for Athena tables
2. **Wrong database** — Using wrong Athena database name
3. **No data yet** — Data takes time to appear after starting export
4. **Table schema changed** — Service manages tables; don't modify manually
5. **Athena workgroup** — Query running in wrong workgroup

## Resolution

- Verify continuous export is ACTIVE
- Find database name in continuous export description
- Tables include: os_info_agent, network_interface_agent, sys_performance_agent, inbound_connection_agent, outbound_connection_agent
- Wait for data to populate after starting continuous export
- Use default Athena workgroup or configure output location
- Example query: `SELECT * FROM os_info_agent LIMIT 10`


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Verify continuous export is ACTIVE | GREEN | Read-only check |
| Find database name | GREEN | Read-only check |
| Wait for data to populate | GREEN | Passive wait |
| Configure Athena workgroup | GREEN | Query configuration |
| Run example queries | GREEN | Read-only data analysis |

## Escalation Conditions

- Discovery agent deployed in production environment

## Data Sensitivity

- **Athena query results**: Server inventory and network data — CONFIDENTIAL
- **Athena output location**: S3 bucket with query results — CONFIDENTIAL
- **Database schema**: Table structure reveals collected data types — INTERNAL

## Prohibited Actions

- NEVER deploy discovery agents without change management approval
- NEVER store IAM access keys in plain text on shared systems
- NEVER modify agent configuration on production servers without a rollback plan

## Phase 3 — Rollback

1. Stop continuous export to halt data flow: `aws discovery stop-continuous-export --export-id <id>`
2. Delete Athena query results from S3 output location
3. Drop Athena tables if no longer needed
4. Delete the Athena database
5. Restrict access to the S3 bucket containing discovery data

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"

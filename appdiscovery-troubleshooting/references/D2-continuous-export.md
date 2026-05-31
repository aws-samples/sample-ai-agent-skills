# D2 — Continuous Export Issues

## Symptoms
- Continuous export not starting
- Data not appearing in S3
- Athena tables not created

## Triage Commands

```bash
aws discovery describe-continuous-exports --max-results 10
aws discovery start-continuous-export
aws s3 ls s3://<export-bucket>/
```

## Common Causes

1. **Not started** — Must explicitly start continuous export
2. **IAM permissions** — Service needs S3 and Athena permissions
3. **Service-linked role** — Missing AWSServiceRoleForApplicationDiscoveryServiceContinuousExport
4. **S3 bucket issues** — Bucket doesn't exist or wrong permissions
5. **Already running** — Can't start if already active

## Resolution

- Start: `aws discovery start-continuous-export`
- Check status: `aws discovery describe-continuous-exports`
- Verify service-linked role exists
- Continuous export creates S3 bucket and Athena database automatically
- Status should be ACTIVE for data to flow
- Stop with: `aws discovery stop-continuous-export --export-id <id>`


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Start continuous export | YELLOW | Creates S3 bucket and Athena resources |
| Check status | GREEN | Read-only check |
| Verify service-linked role | GREEN | Read-only check |
| Stop continuous export | GREEN | Stops data flow |

## Escalation Conditions

- Discovery agent deployed in production environment

## Data Sensitivity

- **Continuous export data**: Ongoing server and network data in S3 — CONFIDENTIAL
- **Athena database**: Queryable discovery data — CONFIDENTIAL
- **Service-linked role**: IAM role with S3 and Athena access — INTERNAL

## Prohibited Actions

- NEVER deploy discovery agents without change management approval
- NEVER store IAM access keys in plain text on shared systems
- NEVER modify agent configuration on production servers without a rollback plan

## Phase 3 — Rollback

1. Stop continuous export: `aws discovery stop-continuous-export --export-id <id>`
2. Verify export status is INACTIVE
3. Delete Athena database and tables if no longer needed
4. Delete S3 bucket contents and bucket if no longer needed
5. Remove the service-linked role if no longer required

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

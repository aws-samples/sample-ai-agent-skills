---
title: "C1 — Workgroup Configuration"
description: "Diagnose and resolve Athena workgroup configuration issues"
status: active
severity: MEDIUM
triggers:
  - "workgroup"
  - "engine version"
  - "data scan limit"
  - "workgroup disabled"
  - "enforce configuration"
owner: devops-agent
objective: "Identify workgroup misconfiguration and correct settings"
context: "Workgroups control Athena behavior including engine version, result location, encryption, data scan limits, and query concurrency. Misconfiguration affects all queries in the workgroup."
---

## Phase 1 — Triage

MUST:
- Get workgroup details: `aws athena get-work-group --work-group <workgroup-name>`
- List all workgroups: `aws athena list-work-groups --query 'WorkGroups[*].{Name:Name,State:State,Engine:EngineVersion.SelectedEngineVersion}'`
- Check if workgroup enforces configuration: `aws athena get-work-group --work-group <workgroup-name> --query 'WorkGroup.Configuration.EnforceWorkGroupConfiguration'`
- Check data scan limit: `aws athena get-work-group --work-group <workgroup-name> --query 'WorkGroup.Configuration.BytesScannedCutoffPerQuery'`

SHOULD:
- Verify result configuration (S3 location, encryption): `aws athena get-work-group --work-group <workgroup-name> --query 'WorkGroup.Configuration.ResultConfiguration'`
- Check engine version compatibility with queries
- Verify IAM permissions for workgroup access

MAY:
- Compare with other working workgroups
- Check CloudTrail for recent workgroup changes

## Phase 2 — Remediate

MUST:
- If workgroup disabled: enable it via `aws athena update-work-group --work-group <name> --state ENABLED`
- If engine version wrong: update via `aws athena update-work-group --work-group <name> --configuration-updates EngineVersion={SelectedEngineVersion=Athena engine version 3}`
- If data scan limit too low: increase or remove the limit
- If result location missing: configure S3 output location

SHOULD:
- Set appropriate data scan limits for cost control
- Enable query result reuse where applicable
- Configure encryption for query results

MAY:
- Create separate workgroups for different teams or use cases
- Set up CloudWatch metrics publishing for workgroup monitoring

## Common Issues

- symptoms: "Query exceeds data scan limit and is cancelled"
  diagnosis: "BytesScannedCutoffPerQuery is set too low for the query."
  resolution: "Increase the limit or optimize the query to scan less data."

- symptoms: "Workgroup shows DISABLED state"
  diagnosis: "Workgroup was disabled manually or by automation."
  resolution: "Enable with aws athena update-work-group --work-group <name> --state ENABLED."

## Output Format

```yaml
root_cause: "Workgroup configuration — <specific_issue>"
evidence:
  - type: workgroup_config
    content: "<workgroup settings>"
severity: MEDIUM
mitigation:
  immediate: "Fix the workgroup configuration setting"
  long_term: "Use IaC to manage workgroup configurations with validated settings"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Enable disabled workgroup | YELLOW | Affects all users of the workgroup |
| Update engine version | YELLOW | May change query behavior for existing workloads |
| Increase/remove data scan limit | YELLOW | Cost control change — may increase spend |
| Configure S3 output location | YELLOW | Changes where query results are stored |
| Set data scan limits for cost control | GREEN | Adds cost guardrail — protective measure |
| Enable query result reuse | GREEN | Performance optimization — non-destructive |

## Escalation Conditions

- Query accesses sensitive data (PII, financial, healthcare records)
- Fix requires modifying workgroup settings that affect multiple teams
- Engine version changes may break existing queries
- Data scan limit changes affect cost controls

## Data Sensitivity

- Workgroup configuration reveals result storage locations and encryption settings
- Engine version changes may affect query result precision
- Data scan limits protect against accidental exposure of large datasets

## Prohibited Actions

- NEVER suggest scanning entire tables without LIMIT to debug — this can incur massive costs and expose sensitive data
- NEVER remove data scan limits without cost management approval
- NEVER change workgroup encryption settings without security team review
- NEVER disable workgroup configuration enforcement without understanding implications

## Phase 3 — Rollback

- If workgroup was enabled incorrectly: disable with `aws athena update-work-group --state DISABLED`
- If engine version was changed: revert using `aws athena update-work-group` with previous version
- If data scan limit was modified: restore original limit value
- If result location was changed: revert to original S3 location
- Document all workgroup configuration changes for audit trail

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
  - "NEVER suggest disabling query result encryption"
  - "NEVER suggest overly broad S3 access for workgroups"
  - "NEVER suggest disabling workgroup enforcement"

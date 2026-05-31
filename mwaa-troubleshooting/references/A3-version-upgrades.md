---
title: "A3 — MWAA Airflow Version Upgrades"
description: "Diagnose issues with MWAA Airflow version upgrades"
status: active
severity: HIGH
triggers:
  - "version upgrade failed"
  - "upgrade airflow"
  - "migration error"
  - "incompatible after upgrade"
owner: devops-agent
objective: "Identify and resolve issues during or after an Airflow version upgrade on MWAA"
context: "MWAA supports in-place Airflow version upgrades. Upgrades may break DAGs using deprecated operators, import paths, or configuration settings. The upgrade process takes 20-30 minutes and requires careful pre-upgrade testing."
---

## Phase 1 — Triage

MUST:
- Check current environment version: `aws mwaa get-environment --name <env-name> --query 'Environment.AirflowVersion'`
- Check environment status: `aws mwaa get-environment --name <env-name> --query 'Environment.{Status:Status,StatusReason:StatusReason}'`
- Check DAG processing errors post-upgrade: `aws logs filter-log-events --log-group-name airflow-<env-name>-DAGProcessing --filter-pattern "ERROR" --start-time <upgrade-epoch>`
- Check scheduler logs for import errors: `aws logs filter-log-events --log-group-name airflow-<env-name>-Scheduler --filter-pattern "ImportError" --start-time <upgrade-epoch>`
- Verify requirements.txt compatibility: `aws s3 cp s3://<bucket>/<requirements-path> -`

SHOULD:
- Check for deprecated operator imports in DAG files: `aws s3 cp s3://<bucket>/<dags-path>/<dag-file> -`
- Review Airflow migration guide for breaking changes between versions
- Check if plugins.zip uses deprecated APIs

MAY:
- Compare DAG parse times before and after upgrade
- Test DAGs in a separate environment with the target version

## Phase 2 — Remediate

MUST:
- Fix deprecated import paths in DAGs (e.g., `airflow.operators` → `airflow.providers`)
- Update requirements.txt to use packages compatible with the new Airflow version
- Update plugins.zip if using deprecated plugin APIs

SHOULD:
- Use Airflow constraints file matching the target version for requirements
- Test all DAGs in a staging environment before upgrading production
- Update connection types if they changed between versions

MAY:
- Create a new environment with the target version for parallel testing
- Roll back by creating a new environment with the previous version if upgrade is unrecoverable

## Common Issues

- symptoms: "DAGs fail to parse after upgrade with ImportError"
  diagnosis: "DAGs use deprecated import paths removed in the new Airflow version."
  resolution: "Update imports to use provider packages (e.g., airflow.providers.amazon.aws.operators)."

- symptoms: "requirements.txt packages fail after upgrade"
  diagnosis: "Package versions incompatible with new Airflow/Python version."
  resolution: "Update requirements.txt with version pins compatible with the new Airflow version."

- symptoms: "Environment stuck in UPDATING during version upgrade"
  diagnosis: "Upgrade process taking longer than expected or blocked by startup errors."
  resolution: "Wait up to 60 minutes. If still stuck, check CloudWatch logs for startup errors."

## Output Format

```yaml
root_cause: "version_upgrade_issue — <specific_cause>"
evidence:
  - type: environment_version
    content: "<current version and target version>"
  - type: dag_errors
    content: "<import errors or parse failures>"
  - type: requirements
    content: "<incompatible packages>"
severity: HIGH
mitigation:
  immediate: "Fix deprecated imports and incompatible packages"
  long_term: "Maintain a staging environment for pre-upgrade testing"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — version upgrades do not directly expose data
- Service Disruption Risk: HIGH — incompatible DAGs or plugins may break after upgrade
- Compliance Impact: MEDIUM — new versions may change security defaults

## Escalation Conditions
- Upgrade causes environment to become UNAVAILABLE
- DAGs fail to parse after version upgrade due to API changes
- Custom plugins incompatible with new Airflow version
- Upgrade path not supported (skipping major versions)
- Performance degradation observed after upgrade

## Data Sensitivity
- Airflow metadata database contents — CONFIDENTIAL
- DAG source code — INTERNAL
- Plugin source code — INTERNAL
- Connection and variable configurations — CONFIDENTIAL
- Never expose Airflow webserver credentials

## Prohibited Actions
- Do NOT skip major Airflow version upgrades (must follow supported upgrade path)
- Do NOT upgrade without testing DAGs and plugins in a staging environment first
- Do NOT upgrade during active critical DAG runs
- Do NOT remove deprecated operator imports without replacing them
- Do NOT disable logging during upgrade to hide errors
- Do NOT upgrade MWAA environments across multiple major versions simultaneously

## Phase 3 — Rollback
- MWAA does not support version downgrades — plan accordingly before upgrading
- If upgrade fails, create a new environment on the previous version and migrate DAGs
- Restore requirements.txt to previous version-compatible dependencies
- Revert plugins.zip to version compatible with the previous Airflow release
- Maintain a backup of the Airflow metadata database before upgrade
- Document incompatibilities found for future upgrade planning

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

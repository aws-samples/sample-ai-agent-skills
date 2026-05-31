---
title: "B1 — MWAA DAG Import Errors"
description: "Diagnose DAG import and parse failures in MWAA"
status: active
severity: HIGH
triggers:
  - "DAG import error"
  - "DAG not showing"
  - "broken DAG"
  - "parse error"
  - "DAG not found"
owner: devops-agent
objective: "Identify why DAGs fail to import or parse and fix the underlying issue"
context: "MWAA syncs DAGs from S3 and the scheduler parses them periodically. Import errors prevent DAGs from appearing in the UI. Common causes include syntax errors, missing dependencies, incorrect imports, and module not found errors."
---

## Phase 1 — Triage

MUST:
- Check DAG processing logs: `aws logs filter-log-events --log-group-name airflow-<env-name>-DAGProcessing --filter-pattern "ERROR" --start-time <epoch-ms>`
- Verify DAG file exists in S3: `aws s3 ls s3://<bucket>/<dags-path>/<dag-file>`
- Check DAG file content for syntax: `aws s3 cp s3://<bucket>/<dags-path>/<dag-file> - | head -50`
- List all DAGs via CLI: use `aws mwaa create-cli-token --name <env-name>` then call `dags list` via the Airflow CLI endpoint
- Check scheduler logs for import errors: `aws logs filter-log-events --log-group-name airflow-<env-name>-Scheduler --filter-pattern "ImportError" --start-time <epoch-ms>`

SHOULD:
- Check if required packages are in requirements.txt: `aws s3 cp s3://<bucket>/<requirements-path> -`
- Verify DAG file uses correct Airflow provider imports for the installed version
- Check for circular imports or top-level code that fails at parse time

MAY:
- Test DAG parsing locally with the same Airflow version
- Check if the DAG file was recently modified: `aws s3api head-object --bucket <bucket> --key <dags-path>/<dag-file>`

## Phase 2 — Remediate

MUST:
- Fix syntax errors in the DAG file
- Add missing packages to requirements.txt and update the environment
- Correct import paths to match the installed Airflow version

SHOULD:
- Avoid top-level code that makes network calls or database queries during DAG parsing
- Use lazy imports for heavy dependencies
- Validate DAG files locally before uploading to S3

MAY:
- Add DAG file validation to CI/CD pipeline
- Use `airflow dags test` locally before deploying

## Common Issues

- symptoms: "DAG not appearing in Airflow UI"
  diagnosis: "DAG file has a parse error preventing import. Check DAGProcessing logs."
  resolution: "Fix the syntax or import error in the DAG file and re-upload to S3."

- symptoms: "ModuleNotFoundError in DAG processing logs"
  diagnosis: "DAG imports a package not in requirements.txt or plugins.zip."
  resolution: "Add the missing package to requirements.txt and update the environment."

- symptoms: "DAG appears as broken in UI with import error"
  diagnosis: "DAG file has a runtime error during parsing (e.g., missing variable, bad connection)."
  resolution: "Fix the top-level code that fails during DAG parsing. Avoid runtime calls at parse time."

## Output Format

```yaml
root_cause: "dag_import_error — <specific_cause>"
evidence:
  - type: dag_processing_logs
    content: "<error messages from DAGProcessing log group>"
  - type: dag_file
    content: "<relevant DAG file content>"
  - type: requirements
    content: "<missing packages>"
severity: HIGH
mitigation:
  immediate: "Fix the DAG file error and re-upload to S3"
  long_term: "Add DAG validation to CI/CD pipeline before S3 upload"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — DAG import errors are typically code-level issues
- Service Disruption Risk: MEDIUM — broken DAGs prevent scheduled workflows from running
- Compliance Impact: LOW — import errors do not directly affect compliance posture

## Escalation Conditions
- All DAGs fail to import simultaneously after S3 sync
- Import errors caused by MWAA platform issue rather than DAG code
- DAG import failures affect production-critical workflows
- Import errors persist after fixing identified syntax or dependency issues
- Scheduler becomes unresponsive due to repeated import failures

## Data Sensitivity
- DAG source code — INTERNAL
- Python tracebacks may contain file paths — INTERNAL
- S3 bucket structure — INTERNAL
- Airflow variable references in DAGs — CONFIDENTIAL
- Never share DAG code containing hardcoded credentials

## Prohibited Actions
- Do NOT upload untested DAG files directly to the production S3 bucket
- Do NOT modify DAGs in S3 without version control
- Do NOT ignore import errors on production-critical DAGs
- Do NOT add broad wildcard imports that may conflict with Airflow internals
- Do NOT bypass DAG validation checks before deployment
- Do NOT store secrets or credentials directly in DAG files

## Phase 3 — Rollback
- Restore previous DAG files from S3 versioned objects
- Remove or rename the broken DAG file in S3 to restore scheduler health
- Revert requirements.txt if a dependency change caused import failures
- Clear the DAG processor manager cache by triggering an environment update if needed
- Verify all previously working DAGs resume normal scheduling after rollback
- Document the root cause of import failure for future prevention

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

---
title: "E1 — MWAA Custom Plugin Issues"
description: "Diagnose custom plugin loading and execution failures in MWAA"
status: active
severity: MEDIUM
triggers:
  - "plugin not loading"
  - "custom operator not found"
  - "plugins.zip error"
  - "plugin import error"
owner: devops-agent
objective: "Identify why custom plugins fail to load or execute in MWAA"
context: "MWAA loads plugins from a plugins.zip file in S3. The ZIP is extracted to the Airflow plugins directory. Incorrect ZIP structure, missing dependencies, or incompatible code causes plugin failures."
---

## Phase 1 — Triage

MUST:
- Check plugins.zip path: `aws mwaa get-environment --name <env-name> --query 'Environment.PluginsS3Path'`
- Verify plugins.zip exists: `aws s3 ls s3://<bucket>/<plugins-path>`
- Check scheduler logs for plugin errors: `aws logs filter-log-events --log-group-name airflow-<env-name>-Scheduler --filter-pattern "plugin" --start-time <epoch-ms>`
- Check DAG processing logs for import errors: `aws logs filter-log-events --log-group-name airflow-<env-name>-DAGProcessing --filter-pattern "ImportError" --start-time <epoch-ms>`
- Verify plugins.zip version: `aws s3api head-object --bucket <bucket> --key <plugins-path>`

SHOULD:
- Download and inspect plugins.zip structure locally
- Verify plugin files are at the correct directory level in the ZIP
- Check if plugin dependencies are in requirements.txt

MAY:
- Test plugin loading locally with the same Airflow version
- Check if plugin conflicts with built-in Airflow plugins

## Phase 2 — Remediate

MUST:
- Fix plugins.zip directory structure — plugin files should be at the ZIP root level
- Add missing plugin dependencies to requirements.txt
- Fix import paths in plugin code

SHOULD:
- Test plugins locally before uploading to S3
- Use Airflow's plugin manager conventions for custom operators/hooks/sensors
- Version plugins.zip in S3 for rollback capability

MAY:
- Split complex plugins into separate modules
- Add plugin unit tests to CI/CD pipeline

## Common Issues

- symptoms: "Custom operator not found when DAG runs"
  diagnosis: "plugins.zip has incorrect directory structure — files nested in extra directory."
  resolution: "Restructure ZIP so plugin files are at the root level, not inside a subdirectory."

- symptoms: "Plugin loads but fails with ImportError"
  diagnosis: "Plugin depends on a package not in requirements.txt."
  resolution: "Add the missing dependency to requirements.txt and update the environment."

- symptoms: "Plugin worked before but broke after environment update"
  diagnosis: "Airflow version upgrade changed plugin API or import paths."
  resolution: "Update plugin code to be compatible with the new Airflow version."

## Output Format

```yaml
root_cause: "custom_plugin_issue — <specific_cause>"
evidence:
  - type: plugins_zip
    content: "<ZIP structure and contents>"
  - type: scheduler_logs
    content: "<plugin loading errors>"
  - type: requirements
    content: "<missing dependencies>"
severity: MEDIUM
mitigation:
  immediate: "Fix plugin ZIP structure or dependencies and update environment"
  long_term: "Add plugin validation and testing to CI/CD pipeline"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — plugins may contain or access sensitive configurations
- Service Disruption Risk: MEDIUM — broken plugins can prevent environment from starting
- Compliance Impact: LOW — plugin issues are typically operational

## Escalation Conditions
- Plugin causes environment to fail during startup
- Plugin introduces security vulnerability
- Plugin conflicts with core Airflow functionality
- Plugin causes memory leaks or performance degradation
- Custom operator in plugin fails across all DAGs using it

## Data Sensitivity
- Plugin source code — INTERNAL
- Plugin configuration parameters — INTERNAL (may contain CONFIDENTIAL data)
- Third-party library dependencies — INTERNAL
- Plugin access credentials — RESTRICTED
- Never include hardcoded credentials in plugin code

## Prohibited Actions
- Do NOT deploy untested plugins directly to production environments
- Do NOT include credentials or secrets in plugin source code
- Do NOT use plugins that require root or elevated system access
- Do NOT bypass plugin packaging requirements (plugins.zip format)
- Do NOT install plugins from untrusted or unverified sources
- Do NOT modify core Airflow behavior through plugins without thorough testing

## Phase 3 — Rollback
- Restore previous plugins.zip from S3 versioned objects
- Remove the problematic plugin and redeploy plugins.zip
- Trigger environment update to reload plugins
- Verify all DAGs using the plugin resume normal operation
- Test plugin in a staging environment before redeploying to production
- Document plugin failure and update plugin testing procedures

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

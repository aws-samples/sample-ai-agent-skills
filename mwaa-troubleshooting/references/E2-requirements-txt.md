---
title: "E2 — MWAA requirements.txt Issues"
description: "Diagnose requirements.txt package installation failures in MWAA"
status: active
severity: HIGH
triggers:
  - "requirements.txt error"
  - "package install failed"
  - "pip install error"
  - "dependency conflict"
  - "environment unhealthy after update"
owner: devops-agent
objective: "Identify and resolve requirements.txt package installation failures that affect MWAA environment health"
context: "MWAA installs packages from requirements.txt at environment startup. Failed installations can make the environment unhealthy. Version conflicts, missing wheels, and network issues are common causes."
---

## Phase 1 — Triage

MUST:
- Check requirements.txt path: `aws mwaa get-environment --name <env-name> --query 'Environment.RequirementsS3Path'`
- Download and review requirements.txt: `aws s3 cp s3://<bucket>/<requirements-path> -`
- Check environment status: `aws mwaa get-environment --name <env-name> --query 'Environment.{Status:Status,StatusReason:StatusReason}'`
- Check worker logs for pip errors: `aws logs filter-log-events --log-group-name airflow-<env-name>-Worker --filter-pattern "pip" --start-time <epoch-ms>`
- Check scheduler logs for startup errors: `aws logs filter-log-events --log-group-name airflow-<env-name>-Scheduler --filter-pattern "ERROR" --start-time <epoch-ms>`

SHOULD:
- Verify all packages have pinned versions
- Check for known incompatibilities with the Airflow version
- Verify constraints file is referenced if used

MAY:
- Test requirements.txt locally with the same Python and Airflow version
- Check S3 object versions for previous working requirements.txt

## Phase 2 — Remediate

MUST:
- Pin all package versions explicitly (e.g., `pandas==1.5.3`)
- Use Airflow constraints file: `--constraint "https://raw.githubusercontent.com/apache/airflow/constraints-<version>/constraints-<python-version>.txt"`
- Revert to previous working requirements.txt if environment is unhealthy

SHOULD:
- Test requirements locally before applying: `pip install -r requirements.txt --constraint <constraints-url>`
- Remove unnecessary packages to minimize installation time and conflicts
- Use S3 versioning to maintain rollback capability

MAY:
- Pre-build wheels for packages that require compilation
- Split requirements into base and optional groups

## Common Issues

- symptoms: "Environment stuck in UPDATING or unhealthy after requirements change"
  diagnosis: "A package in requirements.txt fails to install due to version conflict or missing dependency."
  resolution: "Revert to previous requirements.txt, pin versions, and use constraints file."

- symptoms: "Package installed but import fails at runtime"
  diagnosis: "Package version incompatible with the Airflow or Python version."
  resolution: "Check package compatibility matrix and pin a compatible version."

- symptoms: "Timeout during package installation"
  diagnosis: "Large package or slow network — NAT gateway throughput or PyPI latency."
  resolution: "Verify NAT gateway is healthy. Consider using fewer or smaller packages."

## Output Format

```yaml
root_cause: "requirements_txt_issue — <specific_cause>"
evidence:
  - type: requirements_file
    content: "<requirements.txt contents>"
  - type: pip_logs
    content: "<installation error messages>"
  - type: environment_status
    content: "<environment health status>"
severity: HIGH
mitigation:
  immediate: "Revert to working requirements.txt and stabilize environment"
  long_term: "Pin all versions, use constraints, test locally before deploying"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — requirements files contain package names and versions
- Service Disruption Risk: HIGH — bad requirements can prevent environment from starting
- Compliance Impact: LOW — dependency issues are operational concerns

## Escalation Conditions
- Requirements installation causes environment startup failure
- Dependency conflict prevents environment from becoming available
- Security vulnerability discovered in an installed package
- Requirements installation exceeds timeout limits
- Package installation fails due to network or repository access issues

## Data Sensitivity
- Package names and versions — PUBLIC
- Private PyPI repository URLs — INTERNAL
- Authentication tokens for private repositories — RESTRICTED
- Dependency resolution logs — INTERNAL
- Never expose private repository credentials in requirements.txt

## Prohibited Actions
- Do NOT use unpinned package versions in production requirements.txt
- Do NOT install packages from untrusted PyPI repositories
- Do NOT include packages that conflict with MWAA pre-installed dependencies
- Do NOT exceed the MWAA requirements.txt size limits
- Do NOT use requirements.txt to install system-level packages
- Do NOT deploy requirements changes without testing in a staging environment

## Phase 3 — Rollback
- Restore previous requirements.txt from S3 versioned objects
- Trigger environment update to reinstall previous dependencies
- Pin problematic package to last known working version
- Remove the conflicting package and find an alternative
- Verify environment becomes AVAILABLE after requirements rollback
- Document dependency conflicts for future reference

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

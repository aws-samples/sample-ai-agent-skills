---
title: "A1 — Blu Age Refactoring Errors"
description: "Diagnose Blu Age automated refactoring and code conversion errors"
status: active
severity: HIGH
triggers: ["Blu Age conversion error", "refactoring failed", "COBOL to Java error", "code generation failure"]
owner: devops-agent
objective: "Resolve Blu Age refactoring errors during COBOL-to-Java conversion"
context: "Blu Age converts COBOL source code to Java. Conversion errors can stem from unsupported COBOL constructs, missing copybooks, or complex program structures."
---

## Phase 1 — Triage

MUST:
- Check application status: `aws m2 get-application --application-id <app-id> --query '{Name:name,Status:status,EngineType:engineType}'`
- Review deployment logs: `aws logs filter-log-events --log-group-name /aws/m2/<environment-name> --filter-pattern "ERROR" --start-time <epoch-ms>`
- List recent deployments: `aws m2 list-deployments --application-id <app-id> --query 'deployments[*].{Id:deploymentId,Status:status}'`

SHOULD:
- Check if the COBOL source uses unsupported constructs
- Verify all copybooks are included in the conversion package
- Review the application definition JSON for correct artifact paths

MAY:
- Compare conversion output with expected Java structure
- Check S3 for conversion artifacts and logs

## Phase 2 — Remediate

MUST:
- Ensure all COBOL source files and copybooks are included in the S3 package
- Verify the application definition JSON references correct S3 paths
- Fix unsupported COBOL constructs before re-running conversion

SHOULD:
- Test conversion with a subset of programs to isolate failures
- Review Blu Age documentation for supported COBOL dialects

MAY:
- Engage AWS Professional Services for complex conversion issues
- Implement incremental conversion strategy for large codebases

## Common Issues

- symptoms: "Deployment fails with conversion error for specific COBOL program"
  diagnosis: "COBOL program uses constructs not supported by Blu Age converter."
  resolution: "Review the specific error in deployment logs. Modify COBOL source to use supported constructs. Verify copybooks are complete."

## Safety Ratings

```
safety_ratings:
  - "get-application: GREEN — read-only application check"
  - "logs filter-log-events: GREEN — read-only log review"
  - "list-deployments: GREEN — read-only deployment history"
  - "create-deployment: YELLOW — deploys new application version"
```

## Escalation Conditions

- Conversion fails for critical business programs with no workaround
- COBOL dialect not supported by Blu Age converter
- Conversion produces Java code with incorrect business logic
- Large-scale conversion (1000+ programs) with high failure rate

## Data Sensitivity

- Application source code: HIGH — contains business logic and intellectual property
- Conversion logs: MEDIUM — may reveal code structure and business rules
- Deployment artifacts: HIGH — contain compiled application code
- Copybooks: HIGH — define data structures and business data formats

## Prohibited Actions

- NEVER deploy converted Java code to production without thorough testing
- NEVER modify generated Java code without understanding the impact on reconversion
- NEVER delete COBOL source after conversion — retain for reference and reconversion
- NEVER assume conversion output is correct without functional testing

## Phase 3 — Rollback

1. If a bad deployment was made, redeploy the previous version: `aws m2 create-deployment --application-id <app-id> --application-version <previous-version> --environment-id <env-id>`
2. If conversion artifacts are corrupted, re-upload clean source to S3
3. Restore previous application definition if it was modified
4. Verify the application is functioning with the rolled-back version
5. Document the conversion error and root cause

## Output Format

```yaml
root_cause: "blu_age_refactoring — <specific_cause>"
evidence:
  - type: conversion_error
    content: "<error details from logs>"
severity: HIGH
mitigation:
  immediate: "Fix conversion error"
  long_term: "Implement conversion testing pipeline"
```

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

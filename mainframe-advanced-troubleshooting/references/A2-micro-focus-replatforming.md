---
title: "A2 — Micro Focus Replatforming"
description: "Diagnose Micro Focus replatforming and native COBOL execution issues"
status: active
severity: HIGH
triggers: ["Micro Focus error", "COBOL compilation failed", "replatforming issue", "Micro Focus runtime error"]
owner: devops-agent
objective: "Resolve Micro Focus replatforming issues for native COBOL execution on AWS"
context: "Micro Focus runs COBOL natively on AWS without code conversion. Issues can stem from COBOL compilation, runtime configuration, CICS region setup, or environment configuration."
---

## Phase 1 — Triage

MUST:
- Check environment status: `aws m2 get-environment --environment-id <env-id> --query '{Name:name,Status:status,EngineType:engineType,EngineVersion:engineVersion}'`
- Review application logs: `aws logs filter-log-events --log-group-name /aws/m2/<environment-name> --filter-pattern "ERROR" --start-time <epoch-ms>`
- Check application status: `aws m2 get-application --application-id <app-id>`

SHOULD:
- Verify COBOL compiler directives are correct for the target platform
- Check CICS region configuration if using CICS transactions
- Verify runtime configuration files (system definition, region configuration)

MAY:
- Compare with working on-premises Micro Focus configuration
- Review Micro Focus Enterprise Server documentation for platform differences

## Phase 2 — Remediate

MUST:
- Ensure COBOL source compiles cleanly with Micro Focus compiler on AWS
- Verify runtime configuration matches the application requirements
- Fix CICS region definitions if CICS transactions are failing

SHOULD:
- Test with a subset of programs to isolate compilation issues
- Verify file handler configuration for dataset access

MAY:
- Engage Micro Focus support for compiler-specific issues
- Implement automated compilation and testing pipeline

## Common Issues

- symptoms: "COBOL program compiles on mainframe but fails on Micro Focus"
  diagnosis: "COBOL dialect differences or missing compiler directives."
  resolution: "Add appropriate compiler directives for Micro Focus. Check for mainframe-specific extensions not supported on the target platform."

## Safety Ratings

```
safety_ratings:
  - "get-environment: GREEN — read-only environment check"
  - "logs filter-log-events: GREEN — read-only log review"
  - "get-application: GREEN — read-only application check"
  - "create-deployment: YELLOW — deploys new application version"
```

## Escalation Conditions

- COBOL programs fail to compile with no clear error message
- CICS region configuration issues affecting transaction processing
- Runtime errors that do not occur on the source mainframe
- Micro Focus engine version incompatibility with COBOL source

## Data Sensitivity

- COBOL source code: HIGH — contains business logic and intellectual property
- Runtime configuration: MEDIUM — reveals application architecture
- CICS region definitions: MEDIUM — reveals transaction processing setup
- Application logs: MEDIUM — may contain business data in error messages

## Prohibited Actions

- NEVER deploy untested COBOL programs to production environments
- NEVER modify CICS region configuration without a backup of current settings
- NEVER assume mainframe COBOL compiles identically on Micro Focus without testing
- NEVER delete runtime configuration files without documenting their contents

## Phase 3 — Rollback

1. Redeploy previous application version: `aws m2 create-deployment --application-id <app-id> --application-version <previous-version> --environment-id <env-id>`
2. Restore CICS region configuration from backup
3. Revert compiler directive changes if they caused new issues
4. Verify application functionality after rollback
5. Document the replatforming issue and resolution

## Output Format

```yaml
root_cause: "micro_focus_replatforming — <specific_cause>"
evidence:
  - type: runtime_error
    content: "<compilation or runtime error details>"
severity: HIGH
mitigation:
  immediate: "Fix compilation or runtime issue"
  long_term: "Implement automated testing for replatformed applications"
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

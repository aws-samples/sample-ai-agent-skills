---
title: "D1 — Job Definition Errors"
description: "Diagnose job definition configuration errors"
status: active
severity: MEDIUM
triggers:
  - "job definition error"
  - "invalid job definition"
  - "definition registration failed"
  - "container properties"
  - "job def configuration"
owner: devops-agent
objective: "Identify and resolve job definition errors"
context: "Job definition errors include: invalid container properties, resource specification errors, incompatible platform capabilities, or missing required fields."
---

## Phase 1 — Triage

MUST:
- Check job definition: `aws batch describe-job-definitions --job-definitions <name:revision> --query 'jobDefinitions[*].{Name:jobDefinitionName,Rev:revision,Status:status,Type:type,Platform:platformCapabilities}'`
- Verify container image URI is valid
- Check resource specifications (vCPU, memory)
- Verify platform capabilities match CE type (EC2 or FARGATE)

SHOULD:
- Check for Fargate-specific requirements (executionRoleArn, networkConfiguration)
- Verify environment variables and command
- Check mount points and volumes configuration

MAY:
- Compare with previously working revision
- Check for deprecated fields

## Phase 2 — Remediate

MUST:
- Register corrected job definition: `aws batch register-job-definition --job-definition-name <name> --type container --platform-capabilities <EC2|FARGATE> --container-properties '{"image":"<image>","vcpus":2,"memory":4096,"command":["python","script.py"]}'`
- For Fargate: include executionRoleArn and networkConfiguration
- Ensure resource values are valid for the platform

SHOULD:
- Use job definition revisions for versioning
- Deregister broken definitions: `aws batch deregister-job-definition --job-definition <name:revision>`

MAY:
- Implement job definition validation in CI/CD
- Use IaC for job definition management

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Register corrected job definition | GREEN | New revision — non-destructive |
| Deregister broken definitions | YELLOW | Prevents future use — verify no active references |
| Include executionRoleArn for Fargate | GREEN | Required configuration — non-destructive |
| Fix resource values for platform | GREEN | Configuration correction — non-destructive |
| Implement definition validation in CI/CD | GREEN | Process improvement — non-destructive |

## Escalation Conditions

- Compute environment runs production batch jobs
- Job definition errors block all job submissions
- Definition changes affect multiple job queues or workflows
- Platform capability mismatch requires architecture change

## Data Sensitivity

- **Classification: MEDIUM**
- Job definitions reveal container images, commands, and resource requirements
- Environment variables may reference sensitive configuration
- Platform capabilities expose infrastructure architecture
- Container properties reveal application deployment patterns

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest deregistering job definitions that are actively referenced by running jobs
- **NEVER** suggest embedding secrets in job definition environment variables
- **NEVER** suggest using privileged containers without security justification

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Registered new job definition revision | Submit jobs using previous revision number |
| Deregistered broken definition | Cannot undo — re-register if needed |

## Output Format

```yaml
root_cause: "job_definition_error — <specific_cause>"
evidence:
  - type: definition_config
    content: "<job definition details>"
  - type: validation_error
    content: "<specific error>"
severity: MEDIUM
mitigation:
  immediate: "Register corrected job definition"
  long_term: "Implement definition validation and versioning"
```

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
  - "NEVER suggest overly broad compute environment permissions"
  - "NEVER suggest disabling job queue scheduling"
  - "NEVER suggest public subnet placement for compute environments"

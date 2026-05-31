---
title: "B1 — Application Deployment Failures"
description: "Diagnose application deployment failures in M2 environments"
status: active
severity: HIGH
triggers: ["deployment failed", "cannot deploy application", "deployment stuck", "application version error"]
owner: devops-agent
objective: "Resolve application deployment failures in Mainframe Modernization environments"
context: "Deployments package application artifacts from S3 and deploy them to runtime environments. Failures can stem from S3 access, artifact structure, environment status, or application definition errors."
---

## Phase 1 — Triage

MUST:
- Check deployment status: `aws m2 get-deployment --application-id <app-id> --deployment-id <deployment-id>`
- Verify environment is available: `aws m2 get-environment --environment-id <env-id> --query '{Status:status}'`
- Review deployment logs: `aws logs filter-log-events --log-group-name /aws/m2/<environment-name> --filter-pattern "deployment" --start-time <epoch-ms>`

SHOULD:
- Verify S3 bucket and artifact paths in the application definition
- Check IAM role permissions for S3 access and M2 operations
- Verify application definition JSON syntax

MAY:
- Compare with previous successful deployment configuration
- Check S3 object existence and permissions

## Phase 2 — Remediate

MUST:
- Ensure environment is in Available status before deploying
- Verify S3 artifacts exist at the paths specified in the application definition
- Fix IAM role permissions for S3 and M2 access

SHOULD:
- Validate application definition JSON before deployment
- Test deployment in a non-production environment first

MAY:
- Implement deployment automation with pre-flight checks
- Use versioned S3 objects for deployment artifacts

## Common Issues

- symptoms: "Deployment fails with S3 access error"
  diagnosis: "M2 service role lacks S3 read permissions for the artifact bucket."
  resolution: "Update the M2 service role to include s3:GetObject permission for the artifact bucket and paths."

## Safety Ratings

```
safety_ratings:
  - "get-deployment: GREEN — read-only deployment check"
  - "get-environment: GREEN — read-only environment check"
  - "create-deployment: YELLOW — deploys new application version, may affect running workloads"
  - "IAM role modifications: YELLOW — changes service permissions"
```

## Escalation Conditions

- Deployment fails with internal service errors after fixing all prerequisites
- Environment enters Failed state during deployment
- Deployment succeeds but application does not start correctly
- Multiple consecutive deployment failures with different errors

## Data Sensitivity

- Deployment details: LOW — returns deployment metadata and status
- Application definition: MEDIUM — reveals application architecture and S3 paths
- S3 artifacts: HIGH — contain application code and configuration
- IAM role policies: MEDIUM — reveals service permissions

## Prohibited Actions

- NEVER deploy to a production environment without testing in non-production first
- NEVER modify the application definition without backing up the current version
- NEVER grant overly broad S3 permissions (s3:*) to the M2 service role
- NEVER force-stop a running deployment without understanding the impact

## Phase 3 — Rollback

1. If deployment failed, the previous version should still be running — verify: `aws m2 get-application --application-id <app-id>`
2. If the application is in a bad state, redeploy the previous version
3. Restore application definition if it was modified
4. Verify application functionality after rollback
5. Document the deployment failure and root cause

## Output Format

```yaml
root_cause: "deployment_failure — <specific_cause>"
evidence:
  - type: deployment_error
    content: "<deployment status and error details>"
severity: HIGH
mitigation:
  immediate: "Fix deployment issue"
  long_term: "Implement deployment pipeline with pre-flight checks"
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

---
title: "E1 — Deployment Failures"
description: "Diagnose deployment action failures"
status: active
severity: CRITICAL
triggers: ["deployment failed", "deploy error", "cannot deploy", "deployment timeout"]
owner: devops-agent
objective: "Resolve deployment failures from CodeCatalyst to AWS"
context: "Deployments require AWS connections, IAM roles with trust policies, and properly configured environments."
---

## Phase 1 — Triage

MUST:
- Get workflow run with deployment details: `aws codecatalyst get-workflow-run --space-name <space> --project-name <project> --id <run-id>`
- Verify AWS connection is active in the space
- Check IAM role in the target AWS account: `aws iam get-role --role-name <codecatalyst-deploy-role>`

SHOULD:
- Review deployment action logs for specific errors
- Verify the IAM role trust policy allows CodeCatalyst: `aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument'`
- Check target AWS account resources (CloudFormation stacks, ECS services, etc.)

MAY:
- Review CloudTrail in the target account for deployment API calls
- Check if deployment environment configuration matches target account

## Phase 2 — Remediate

MUST:
- Fix IAM role trust policy to allow codecatalyst.amazonaws.com
- Ensure IAM role has permissions for deployment actions
- Verify AWS connection is properly configured in the space

SHOULD:
- Update deployment action configuration with correct environment and connection
- Fix target resource configuration (CloudFormation template, ECS task definition)
- Verify deployment artifacts are correctly built

MAY:
- Implement deployment rollback automation
- Configure deployment approval gates

## Safety Ratings

```
safety_ratings:
  - "get-workflow-run: GREEN — read-only run details"
  - "iam get-role: GREEN — read-only role details"
  - "deployment re-run: RED — deploys to target environment"
  - "IAM role policy modification: YELLOW — changes deployment permissions"
```

## Escalation Conditions

- Deployment failure causes production outage
- IAM role cannot be modified due to organizational policies
- AWS connection is broken and cannot be re-established
- Deployment rollback also fails

## Data Sensitivity

- `aws codecatalyst get-workflow-run`: LOW — returns run metadata
- `aws iam get-role`: MEDIUM — reveals role configuration and trust relationships
- Deployment logs: MEDIUM — may contain resource identifiers and configuration
- IAM role policies: MEDIUM — reveals deployment permissions

## Prohibited Actions

- NEVER grant AdministratorAccess to deployment IAM roles
- NEVER deploy to production without proper approval workflow
- NEVER modify IAM trust policies without understanding security implications
- NEVER skip deployment validation steps

## Phase 3 — Rollback

1. If deployment partially completed, roll back the target resources to previous state
2. If IAM role was modified, restore original policy
3. Verify target environment is in a healthy state after rollback
4. Confirm the previous deployment version is serving traffic
5. Document the deployment failure and rollback steps

## Common Issues

- symptoms: "Deployment fails with 'AssumeRole failed'"
  diagnosis: "IAM role trust policy does not allow CodeCatalyst service principal."
  resolution: "Update IAM role trust policy to include codecatalyst.amazonaws.com as trusted principal."

- symptoms: "Deployment fails with 'AccessDenied' on AWS resource"
  diagnosis: "IAM role lacks permissions for the specific deployment action."
  resolution: "Add required permissions to the IAM role policy (e.g., cloudformation:*, ecs:*, s3:*)."

## Output Format

```yaml
root_cause: "deployment_failure — <specific_cause>"
evidence:
  - type: deployment_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix deployment and redeploy"
  long_term: "Implement deployment validation and rollback automation"
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

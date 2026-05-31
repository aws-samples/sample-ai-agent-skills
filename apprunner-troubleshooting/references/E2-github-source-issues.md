---
title: "E2 — App Runner GitHub Source Issues"
description: "Diagnose GitHub source connection, build configuration, and automatic deployment problems"
status: active
severity: MEDIUM
triggers:
  - "GitHub source"
  - "GitHub connection"
  - "source code"
  - "code repository"
  - "branch not found"
  - "connection failed"
  - "Bitbucket"
owner: devops-agent
objective: "Identify and fix GitHub/Bitbucket source connection and configuration issues"
context: "Code-based sources use a GitHub or Bitbucket connection to pull source code, build a container, and deploy it. The connection must be in AVAILABLE status. Build configuration can come from the API/console or an apprunner.yaml file in the repository. Common issues include connection authorization, branch configuration, build command errors, and automatic deployment triggers."
---

## Phase 1 — Triage

MUST:
- Check code repository configuration: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.SourceConfiguration.CodeRepository'`
- Check connection status: `aws apprunner list-connections --query 'ConnectionSummaryList[*].{Name:ConnectionName,Arn:ConnectionArn,Status:Status}'`
- Check auto-deployment setting: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.SourceConfiguration.AutoDeploymentsEnabled'`
- Review build and service logs: `aws logs get-log-events --log-group-name /aws/apprunner/<service-name>/<service-id>/service --log-stream-name <stream>`
- List recent operations: `aws apprunner list-operations --service-arn <service-arn>`

SHOULD:
- Verify the connection is in AVAILABLE status (not PENDING_HANDSHAKE or ERROR)
- Check if the configured branch exists in the repository
- Verify the code configuration source (API vs REPOSITORY for apprunner.yaml)

MAY:
- Check if the GitHub App installation has access to the specific repository
- Verify the apprunner.yaml file format if using repository-based configuration
- Check GitHub webhook delivery status for automatic deployment triggers

## Phase 2 — Remediate

MUST:
- If connection is PENDING_HANDSHAKE: complete the OAuth handshake in the AWS console
- If branch not found: update the service with the correct branch name
- If build fails: fix build commands or apprunner.yaml configuration (see A2)

SHOULD:
- Use the AWS console to create and authorize GitHub connections (OAuth flow required)
- Ensure the GitHub App has access to the specific repository (not just the organization)
- Use API-based configuration for simpler management, or apprunner.yaml for repository-driven config

MAY:
- Recreate the connection if it's in an error state
- Set up branch protection rules to prevent broken code from triggering auto-deployments

## Common Issues

- symptoms: "Connection status is PENDING_HANDSHAKE"
  diagnosis: "GitHub OAuth authorization was not completed after creating the connection."
  resolution: "Go to the App Runner console, select the connection, and complete the GitHub authorization flow."

- symptoms: "Automatic deployment not triggering on git push"
  diagnosis: "AutoDeploymentsEnabled is false, or the GitHub App webhook is not configured."
  resolution: "Enable auto-deployments. Verify the GitHub App installation has webhook access to the repository."

- symptoms: "Build fails with 'apprunner.yaml not found'"
  diagnosis: "Code configuration source is set to REPOSITORY but apprunner.yaml is missing from the repo root."
  resolution: "Add apprunner.yaml to the repository root, or switch to API-based configuration."

## Output Format

```yaml
root_cause: "github_source — <specific_cause>"
evidence:
  - type: code_repository_config
    content: "<repository URL, branch, configuration source>"
  - type: connection_status
    content: "<connection name and status>"
  - type: build_logs
    content: "<relevant build log entries>"
severity: MEDIUM
mitigation:
  immediate: "Fix connection, branch, or build configuration"
  long_term: "Implement CI/CD validation before auto-deployment triggers"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Complete OAuth handshake | YELLOW | Authorization flow |
| Update service with correct branch | YELLOW | Source configuration change |
| Fix build commands or apprunner.yaml | YELLOW | Build configuration change |
| Enable auto-deployments | YELLOW | Changes deployment trigger |
| Recreate connection | YELLOW | Connection replacement |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Code repository config**: Repository URL, branch — INTERNAL
- **Connection status**: GitHub/Bitbucket connection details — INTERNAL
- **Build logs**: Build output and errors — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER share GitHub connection credentials or OAuth tokens

## Phase 3 — Rollback

1. Revert branch configuration to previous branch
2. Revert apprunner.yaml or build configuration changes
3. Disable auto-deployment if it was enabled and caused issues
4. Trigger manual deployment from the previous known-good branch
5. Verify service is healthy after rollback

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"

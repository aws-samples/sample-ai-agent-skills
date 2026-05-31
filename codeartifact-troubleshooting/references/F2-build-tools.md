---
title: "F2 — Build Tool Configuration Issues"
description: "Diagnose build tool configuration problems with CodeArtifact including Gradle, NuGet, and Swift"
status: active
severity: MEDIUM
triggers:
  - "gradle codeartifact"
  - "nuget codeartifact"
  - "swift codeartifact"
  - "build tool config"
  - "package manager config"
owner: devops-agent
objective: "Resolve build tool configuration issues with CodeArtifact"
context: "Different build tools require specific configuration to work with CodeArtifact. Each tool has its own auth mechanism, config file format, and endpoint requirements. Common issues include wrong endpoint format, missing auth config, and tool-specific quirks."
---

## Phase 1 — Triage

MUST:
- Identify the build tool and package format
- Get correct endpoint: `aws codeartifact get-repository-endpoint --domain <domain> --repository <repo> --format <format>`
- Verify auth token is current
- Check tool-specific configuration files

SHOULD:
- Verify the endpoint URL format matches what the tool expects
- Check for conflicting configuration in multiple config files
- Verify the tool version supports CodeArtifact

MAY:
- Run build with verbose/debug logging
- Check tool documentation for CodeArtifact-specific setup

## Phase 2 — Remediate

MUST:
- Use `aws codeartifact login` with the correct `--tool` flag when available
- For unsupported tools, manually configure with the correct endpoint and token
- Ensure auth token is in the correct format for the tool

SHOULD:
- Use environment variables for token injection where possible
- Document tool-specific configuration for the team

## Common Issues

- symptoms: "Gradle cannot resolve from CodeArtifact"
  diagnosis: "build.gradle repository URL or credentials misconfigured."
  resolution: "Use aws codeartifact login --tool gradle or configure build.gradle with correct endpoint."

- symptoms: "NuGet restore fails"
  diagnosis: "NuGet source not configured or token expired."
  resolution: "Configure NuGet source with CodeArtifact endpoint and fresh token."

## Output Format

```yaml
root_cause: "build_tool_config — <specific_cause>"
evidence:
  - type: tool_config
    content: "<build tool configuration>"
  - type: endpoint
    content: "<CodeArtifact endpoint>"
severity: MEDIUM
mitigation:
  immediate: "Fix build tool configuration"
  long_term: "Create reusable configuration templates for each build tool"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Use aws codeartifact login with correct tool flag | GREEN |
| Manually configure unsupported tools | GREEN |
| Ensure auth token is in correct format | GREEN |
| Use environment variables for token injection | GREEN |
| Document tool-specific configuration | GREEN |

## Escalation Conditions

- Fix requires modifying production build tool configurations
- Fix involves changing build configurations that affect all team members

## Data Sensitivity

- **MEDIUM**: build tool configurations contain repository endpoints and auth credentials
- Tool-specific config files may be committed to source control

## Prohibited Actions

- NEVER suggest deleting upstream connections without understanding dependency impact
- NEVER suggest publishing over existing package versions

## Phase 3 — Rollback

1. If build tool configuration was changed, restore previous config files
2. If environment variables were modified, restore previous values
3. If tool version was changed, revert to previous version
4. Verify rollback by confirming build tool resolves packages correctly

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

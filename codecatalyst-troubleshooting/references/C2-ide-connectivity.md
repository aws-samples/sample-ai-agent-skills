---
title: "C2 — IDE Connectivity"
description: "Diagnose IDE connectivity issues with dev environments"
status: active
severity: MEDIUM
triggers: ["IDE cannot connect", "VS Code connection failed", "JetBrains error", "browser IDE not loading"]
owner: devops-agent
objective: "Resolve IDE connectivity issues with CodeCatalyst dev environments"
context: "Dev environments are accessed through VS Code, JetBrains IDEs, or the browser-based IDE. SSH is not supported."
---

## Phase 1 — Triage

MUST:
- Verify dev environment is running: `aws codecatalyst get-dev-environment --space-name <space> --project-name <project> --id <env-id>`
- Check IDE extension/plugin version is current
- Verify Builder ID authentication in the IDE

SHOULD:
- Check local network connectivity (firewall, proxy, VPN)
- Review IDE extension logs for connection errors
- Verify the CodeCatalyst extension is properly installed

MAY:
- Test connectivity with the browser-based IDE as a fallback
- Check if the issue is specific to one IDE or affects all

## Phase 2 — Remediate

MUST:
- Ensure dev environment status is RUNNING before connecting
- Update IDE extension/plugin to latest version
- Re-authenticate Builder ID in the IDE if session expired

SHOULD:
- Configure proxy settings in IDE if behind corporate proxy
- Allow CodeCatalyst endpoints through firewall
- Restart IDE after extension updates

MAY:
- Use browser-based IDE as temporary workaround
- Clear IDE extension cache and re-install

## Safety Ratings

```
safety_ratings:
  - "get-dev-environment: GREEN — read-only environment status"
  - "IDE extension update: GREEN — non-destructive software update"
  - "Builder ID re-authentication: GREEN — refreshes authentication"
  - "IDE restart: GREEN — non-destructive application restart"
```

## Escalation Conditions

- IDE connectivity fails for all users in the organization
- Dev environment is running but no IDE can connect
- Connection drops repeatedly during active development
- Browser-based IDE also fails to connect

## Data Sensitivity

- `aws codecatalyst get-dev-environment`: LOW — returns environment metadata
- IDE extension logs: LOW — connection diagnostic information
- Builder ID session: MEDIUM — authentication token
- Dev environment content: HIGH — developer work in progress

## Prohibited Actions

- NEVER attempt SSH connections to dev environments
- NEVER share Builder ID session tokens
- NEVER disable TLS/SSL to troubleshoot connectivity
- NEVER use unsupported IDE versions or unofficial extensions

## Phase 3 — Rollback

1. If IDE extension was updated, revert to previous version if new version causes issues
2. If proxy settings were changed, restore original configuration
3. Verify IDE can connect to the dev environment after changes
4. Confirm no work was lost during connectivity interruption
5. Document the connectivity issue and resolution

## Common Issues

- symptoms: "VS Code shows 'Failed to connect to dev environment'"
  diagnosis: "CodeCatalyst extension may be outdated or Builder ID session expired."
  resolution: "Update the CodeCatalyst extension. Re-authenticate with Builder ID."

- symptoms: "IDE connects but terminal/editor is unresponsive"
  diagnosis: "Dev environment may be resource-constrained or network latency is high."
  resolution: "Check dev environment resource usage. Consider upgrading instance type."

## Output Format

```yaml
root_cause: "ide_connectivity — <specific_cause>"
evidence:
  - type: connection_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Restore IDE connectivity"
  long_term: "Document network requirements and IDE setup guide"
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

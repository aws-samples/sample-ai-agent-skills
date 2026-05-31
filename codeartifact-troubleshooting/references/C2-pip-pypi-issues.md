---
title: "C2 — pip/PyPI Issues"
description: "Diagnose pip package resolution and installation failures with CodeArtifact"
status: active
severity: HIGH
triggers:
  - "pip install failed"
  - "pip codeartifact"
  - "pypi codeartifact"
  - "pip 401"
  - "pip package not found"
owner: devops-agent
objective: "Resolve pip/PyPI integration issues with CodeArtifact"
context: "pip requires --index-url configuration to use CodeArtifact. Using --extra-index-url creates dependency confusion risk. Common issues include wrong index URL, expired tokens, and package name mismatches."
---

## Phase 1 — Triage

MUST:
- Check pip configuration: `pip config list`
- Get CodeArtifact PyPI endpoint: `aws codeartifact get-repository-endpoint --domain <domain> --repository <repo> --format pypi`
- Verify auth token is current
- Check if package exists: `aws codeartifact list-packages --domain <domain> --repository <repo> --format pypi`

SHOULD:
- Check for pip.conf or pip.ini overrides
- Verify --index-url is used (NOT --extra-index-url)
- Check upstream connection to public PyPI

MAY:
- Run with verbose: `pip install --verbose <package>`
- Check for environment variable overrides: `PIP_INDEX_URL`, `PIP_EXTRA_INDEX_URL`

## Phase 2 — Remediate

MUST:
- Configure pip with CodeArtifact: `aws codeartifact login --tool pip --domain <domain> --repository <repo>`
- Use `--index-url` not `--extra-index-url` to prevent dependency confusion
- Refresh expired token

SHOULD:
- Configure pip.conf for persistent configuration
- Add upstream connection to public PyPI for public packages

## Output Format

```yaml
root_cause: "pip_pypi_issue — <specific_cause>"
evidence:
  - type: pip_config
    content: "<pip configuration>"
  - type: endpoint
    content: "<CodeArtifact PyPI endpoint>"
severity: HIGH
mitigation:
  immediate: "Configure pip index-url and refresh auth token"
  long_term: "Use pip.conf and automate token refresh. Never use extra-index-url."
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Configure pip with CodeArtifact login | GREEN |
| Use --index-url not --extra-index-url | GREEN |
| Refresh expired token | GREEN |
| Configure pip.conf for persistent configuration | GREEN |
| Add upstream connection to public PyPI | YELLOW |

## Escalation Conditions

- Fix requires modifying production CI/CD pipeline pip configurations
- Fix involves changing pip.conf that affects all team members

## Data Sensitivity

- **MEDIUM**: pip configurations contain registry endpoints and auth tokens
- Using --extra-index-url creates dependency confusion risk exposing internal package names

## Prohibited Actions

- NEVER suggest deleting upstream connections without understanding dependency impact
- NEVER suggest publishing over existing package versions

## Phase 3 — Rollback

1. If pip.conf was modified, restore previous version
2. If pip index-url was changed, restore previous: `pip config set global.index-url <previous-url>`
3. If upstream connection was added, disassociate if causing issues
4. Verify rollback by confirming `pip install` resolves packages correctly

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

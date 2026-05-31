---
title: "A2 — Upstream Connection Issues"
description: "Diagnose CodeArtifact upstream repository and external connection problems"
status: active
severity: HIGH
triggers:
  - "upstream connection"
  - "external connection"
  - "package not found"
  - "upstream resolution"
  - "dependency not resolved"
owner: devops-agent
objective: "Fix upstream connection configuration to enable proper package resolution"
context: "CodeArtifact resolves packages through upstream repositories and external connections to public registries. Common issues include missing external connections, wrong upstream order, and resolution failures."
---

## Phase 1 — Triage

MUST:
- Check repository upstreams: `aws codeartifact describe-repository --domain <domain> --repository <repo> --query 'repository.upstreams'`
- List external connections: `aws codeartifact list-repositories-in-domain --domain <domain>` and check each for external connections
- Verify the package exists in upstream: `aws codeartifact list-packages --domain <domain> --repository <upstream-repo> --query 'packages[?package==`<pkg>`]'`
- Check upstream order — first match wins

SHOULD:
- Verify external connection is to the correct public registry (npmjs, pypi, maven-central)
- Test package resolution: `aws codeartifact list-package-versions --domain <domain> --repository <repo> --package <pkg> --format <format>`
- Check if upstream repository has the package cached

MAY:
- Associate an external connection: `aws codeartifact associate-external-connection --domain <domain> --repository <repo> --external-connection public:<registry>`
- Check if the public registry is experiencing issues

## Phase 2 — Remediate

MUST:
- If no external connection, add one: `aws codeartifact associate-external-connection --domain <domain> --repository <repo> --external-connection public:npmjs`
- If upstream order wrong, update: `aws codeartifact update-repository --domain <domain> --repository <repo> --upstreams repositoryName=<upstream1> repositoryName=<upstream2>`
- If package not in any upstream, verify package name and version exist in public registry

SHOULD:
- Order upstreams: internal repos first, external connection repos last
- Use separate repos for different external connections (one per repo)

## Output Format

```yaml
root_cause: "upstream_connection — <specific_cause>"
evidence:
  - type: repository_config
    content: "<upstream configuration>"
  - type: package_search
    content: "<package resolution result>"
severity: HIGH
mitigation:
  immediate: "Fix upstream connections or ordering"
  long_term: "Document upstream topology and monitor resolution paths"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Add external connection | YELLOW |
| Update upstream order | RED |
| Verify package name and version in public registry | GREEN |
| Order upstreams: internal first, external last | GREEN |
| Use separate repos for different external connections | GREEN |

## Escalation Conditions

- Fix requires modifying production upstream connections
- Fix involves reordering upstreams which affects package resolution for all consumers

## Data Sensitivity

- **HIGH**: upstream configurations control package resolution paths and dependency sources
- External connections expose which public registries are trusted

## Prohibited Actions

- NEVER suggest deleting upstream connections without understanding dependency impact
- NEVER suggest publishing over existing package versions

## Phase 3 — Rollback

1. If upstream order was changed, restore previous order: `aws codeartifact update-repository --domain <domain> --repository <repo> --upstreams <previous-order>`
2. If external connection was added, disassociate if causing issues: `aws codeartifact disassociate-external-connection --domain <domain> --repository <repo> --external-connection <connection>`
3. If upstream was removed, re-add it with correct configuration
4. Verify rollback by confirming package resolution works as expected

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

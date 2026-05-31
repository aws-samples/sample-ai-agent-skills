---
title: "C3 — Maven Issues"
description: "Diagnose Maven dependency resolution and build failures with CodeArtifact"
status: active
severity: HIGH
triggers:
  - "maven build failed"
  - "maven codeartifact"
  - "maven 401"
  - "maven dependency not found"
  - "settings.xml codeartifact"
owner: devops-agent
objective: "Resolve Maven integration issues with CodeArtifact"
context: "Maven requires settings.xml configuration with server credentials and repository definitions. Common issues include missing settings.xml entries, expired auth tokens, wrong repository URLs, and pom.xml conflicts."
---

## Phase 1 — Triage

MUST:
- Check settings.xml for server and repository entries
- Get CodeArtifact Maven endpoint: `aws codeartifact get-repository-endpoint --domain <domain> --repository <repo> --format maven`
- Verify auth token is current
- Check if artifact exists: `aws codeartifact list-packages --domain <domain> --repository <repo> --format maven`

SHOULD:
- Verify settings.xml server ID matches repository/mirror ID
- Check for pom.xml repository definitions that may conflict
- Verify upstream connection to Maven Central

MAY:
- Run with debug: `mvn install -X`
- Check local Maven cache: `~/.m2/repository`

## Phase 2 — Remediate

MUST:
- Configure Maven with CodeArtifact: `aws codeartifact login --tool mvn --domain <domain> --repository <repo>`
- Ensure settings.xml has matching server and repository entries
- Refresh expired auth token before build

SHOULD:
- Use mirrors in settings.xml to redirect all Maven Central requests through CodeArtifact
- Add upstream connection to maven-central for public dependencies

## Output Format

```yaml
root_cause: "maven_issue — <specific_cause>"
evidence:
  - type: settings_xml
    content: "<relevant settings.xml entries>"
  - type: endpoint
    content: "<CodeArtifact Maven endpoint>"
severity: HIGH
mitigation:
  immediate: "Fix settings.xml configuration and refresh auth token"
  long_term: "Automate settings.xml generation and token refresh in CI/CD"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Configure Maven with CodeArtifact login | GREEN |
| Ensure settings.xml has matching entries | GREEN |
| Refresh expired auth token | GREEN |
| Use mirrors in settings.xml | GREEN |
| Add upstream connection to maven-central | YELLOW |

## Escalation Conditions

- Fix requires modifying production CI/CD pipeline Maven configurations
- Fix involves changing settings.xml that affects all team members

## Data Sensitivity

- **MEDIUM**: Maven settings.xml contains server credentials and repository endpoints
- Mirror configurations reveal internal package management topology

## Prohibited Actions

- NEVER suggest deleting upstream connections without understanding dependency impact
- NEVER suggest publishing over existing package versions

## Phase 3 — Rollback

1. If settings.xml was modified, restore previous version from backup
2. If Maven mirror was changed, restore previous mirror configuration
3. If upstream connection was added, disassociate if causing issues
4. Verify rollback by confirming `mvn install` resolves dependencies correctly

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

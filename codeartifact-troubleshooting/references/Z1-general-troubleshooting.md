---
title: "Z1 — General CodeArtifact Troubleshooting (Catch-All)"
description: "Fallback SOP for CodeArtifact issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown CodeArtifact issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- List all domains: `aws codeartifact list-domains`
- List all repositories: `aws codeartifact list-repositories`
- If specific domain/repo involved, get details: `aws codeartifact describe-domain --domain <domain>` and `aws codeartifact describe-repository --domain <domain> --repository <repo>`
- Check recent events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=codeartifact.amazonaws.com --max-results 10`
- Verify authentication: `aws codeartifact get-authorization-token --domain <domain> --query authorizationToken`

SHOULD:
- Check domain and repository permissions policies
- Verify upstream connections and external connections
- Check package manager configuration (npm, pip, Maven)

MAY:
- Check AWS Health Dashboard for CodeArtifact service events
- Check service quotas: `aws service-quotas list-service-quotas --service-code codeartifact`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Repository creation errors → Repository (A1)
- Upstream/external connection issues → Repository (A2)
- Auth token problems → Auth (B1)
- Domain/repo permission denied → Auth (B2)
- npm failures → Package (C1)
- pip/PyPI failures → Package (C2)
- Maven failures → Package (C3)
- Publishing errors → Publishing (D1)
- Version conflicts → Publishing (D2)
- Domain config issues → Domain (E1)
- Cross-account access → Domain (E2)
- CI/CD pipeline failures → Integration (F1)
- Build tool config → Integration (F2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all domain and repository configurations
- Review CloudTrail for recent configuration changes
- Verify network connectivity to CodeArtifact endpoints
- Test basic operations: list, describe, get-authorization-token

SHOULD:
- Compare with a known-good configuration
- Check AWS Health Dashboard for service events

MAY:
- Check for regional service outages
- Review package manager logs for client-side errors

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<repository|auth|package|publishing|domain|integration|unknown>"
investigation_path: "domain status → repository config → auth → <domain_classification>"
evidence:
  - type: domain_config
    content: "<domain configuration summary>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| List all domains and repositories | GREEN |
| Get domain and repository details | GREEN |
| Check recent CloudTrail events | GREEN |
| Verify authentication | GREEN |
| Check domain and repository permissions | GREEN |

## Escalation Conditions

- Issue cannot be classified into any specific failure domain
- Fix requires modifying production domain or repository configurations

## Data Sensitivity

- **MEDIUM**: general troubleshooting may expose package management topology and access patterns
- Investigation may reveal repository structures, upstream connections, and permission policies

## Prohibited Actions

- NEVER suggest deleting upstream connections without understanding dependency impact
- NEVER suggest publishing over existing package versions

## Phase 3 — Rollback

1. If any configuration was changed during investigation, restore previous settings
2. If permissions were modified, revert to previous policies
3. Document all changes made during investigation for audit purposes
4. Verify rollback by confirming all CodeArtifact resources match pre-investigation state

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

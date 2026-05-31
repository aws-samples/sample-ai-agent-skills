---
title: "Z1 — General Amplify Troubleshooting"
description: "Catch-all systematic investigation for Amplify issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "Amplify issue"
  - "Amplify problem"
  - "Amplify not working"
  - "Amplify error"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate Amplify issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for Amplify issues that don't clearly match a specific runbook. It covers app health, build status, deployment status, domain configuration, and systematic elimination of common causes."
---

## Phase 1 — Triage

MUST:
- Check app overview: `aws amplify get-app --app-id <app-id>`
- Check all branches and their status: `aws amplify list-branches --app-id <app-id>`
- Check recent jobs: `aws amplify list-jobs --app-id <app-id> --branch-name <branch> --max-results 10`
- Check domain associations: `aws amplify list-domain-associations --app-id <app-id>`
- Check backend environments: `aws amplify list-backend-environments --app-id <app-id>`

SHOULD:
- Review build logs for the most recent failed job
- Check environment variables at app and branch level
- Verify repository connection is active
- Check CloudWatch logs for runtime errors

MAY:
- Check AWS Health Dashboard for Amplify service issues
- Review IAM service role permissions
- Check for recent configuration changes in CloudTrail

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Build issues → A1 (failures), A2 (settings), A3 (env vars)
  - Deploy issues → B1 (failures), B2 (branch config), B3 (PR previews)
  - Domain issues → C1 (custom domain), C2 (SSL)
  - Backend issues → D1 (environment), D2 (API)
  - SSR issues → E1 (rendering), E2 (compute)
  - Auth issues → F1 (access control), F2 (basic auth)
  - Performance → G1 (CDN caching), G2 (rewrite rules)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Review recent changes that may have caused the issue

MAY:
- Set up comprehensive monitoring
- Create post-incident report

## Safety Ratings

```
safety_ratings:
  - "get-app: GREEN — read-only app overview"
  - "list-branches: GREEN — read-only branch enumeration"
  - "list-jobs: GREEN — read-only job status check"
  - "list-domain-associations: GREEN — read-only domain check"
  - "list-backend-environments: GREEN — read-only backend check"
```

## Escalation Conditions

- Application completely non-functional and cannot be diagnosed
- Issue cannot be classified into any specific failure domain
- Multiple failure domains affected simultaneously
- AWS Health Dashboard shows Amplify service event

## Data Sensitivity

- App configuration: LOW — app metadata and settings
- Build status: LOW — build pass/fail information
- Domain configuration: LOW — domain names and status
- Backend environments: MEDIUM — backend resource ARNs

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER make configuration changes during general troubleshooting without first identifying root cause
- NEVER dismiss the issue without documenting the investigation path

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore app settings, branch configuration, and domain associations to pre-investigation state
3. Verify builds, deployments, and domain resolution work correctly
4. Document the investigation path and findings for the team
5. If the issue remains unresolved, escalate to AWS Support with collected evidence

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: app_status
    content: "<app configuration>"
  - type: build_status
    content: "<recent build results>"
  - type: domain_status
    content: "<domain configuration>"
severity: MEDIUM
general_analysis:
  app_state: "healthy | degraded | failing"
  build_status: "passing | failing"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive monitoring and alerting"
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

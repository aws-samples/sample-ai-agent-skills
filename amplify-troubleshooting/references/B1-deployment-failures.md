---
title: "B1 — Deployment Failures"
description: "Diagnose why Amplify deployments are failing after successful builds"
status: active
severity: HIGH
triggers:
  - "deployment failed"
  - "deploy error"
  - "deployment stuck"
  - "publish failed"
  - "deployment timeout"
owner: devops-agent
objective: "Identify why deployments are failing and resolve the issue"
context: "Deployment failures occur after a successful build when: artifacts are missing or misconfigured, CloudFront distribution update fails, SSR deployment errors, or IAM permission issues."
---

## Phase 1 — Triage

MUST:
- Check job status and steps: `aws amplify get-job --app-id <app-id> --branch-name <branch> --job-id <job-id>`
- Verify build succeeded but deploy failed (check step statuses)
- Check artifacts configuration in build settings
- Review deployment logs for specific errors

SHOULD:
- Verify artifacts baseDirectory points to correct build output
- Check if SSR artifacts are properly generated
- Verify IAM service role has required permissions

MAY:
- Check CloudFront distribution status
- Review if concurrent deployments are conflicting

## Phase 2 — Remediate

MUST:
- Fix artifacts path in amplify.yml or console settings
- Ensure build output directory exists and contains expected files
- Fix IAM permissions if access denied

SHOULD:
- Verify artifacts configuration: baseDirectory should match framework output (build, dist, out, .next)
- Retry deployment: `aws amplify start-job --app-id <app-id> --branch-name <branch> --job-type RETRY`

MAY:
- Check for deployment size limits
- Review CloudFront distribution for errors

## Safety Ratings

```
safety_ratings:
  - "get-job (step statuses): GREEN — read-only job check"
  - "check artifacts configuration: GREEN — read-only config check"
  - "fix artifacts path: YELLOW — changes deployment configuration"
  - "start-job RETRY: YELLOW — triggers new deployment"
```

## Escalation Conditions

- Deployment failures blocking production releases
- CloudFront distribution errors requiring AWS Support
- IAM service role permissions insufficient for deployment
- Concurrent deployment conflicts causing intermittent failures

## Data Sensitivity

- Job status and steps: LOW — deployment pipeline metadata
- Artifacts configuration: LOW — build output paths
- Deployment logs: MEDIUM — may contain error details and resource identifiers

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER retry deployments repeatedly without fixing the underlying issue
- NEVER modify artifacts path without verifying the build output directory exists

## Phase 3 — Rollback

1. If deployment broke the live site, redeploy the previous successful build: `aws amplify start-job --app-id <id> --branch-name <branch> --job-type RETRY --job-id <previous-successful-job-id>`
2. If artifacts path was changed incorrectly, restore the previous path
3. If IAM permissions were changed, restore previous service role policies
4. Verify the site is serving correctly after rollback
5. Document the deployment failure for post-incident review

## Output Format

```yaml
root_cause: "deployment_failure — <specific_cause>"
evidence:
  - type: job_status
    content: "<job step statuses>"
  - type: artifacts
    content: "<artifacts configuration>"
severity: HIGH
mitigation:
  immediate: "Fix artifacts path or IAM permissions"
  long_term: "Implement deployment validation and monitoring"
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

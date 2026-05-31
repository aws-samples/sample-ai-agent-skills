---
title: "D2 — API Integration Issues"
description: "Diagnose API integration problems with Amplify backend"
status: active
severity: MEDIUM
triggers:
  - "API not working"
  - "GraphQL error"
  - "REST API error"
  - "API endpoint not reachable"
  - "backend API failure"
owner: devops-agent
objective: "Identify and resolve API integration issues"
context: "API integration issues include: API endpoint misconfigured, CORS errors, authentication failures, API not deployed, or environment-specific API URLs not set."
---

## Phase 1 — Triage

MUST:
- Check backend environment for API resources: `aws amplify get-backend-environment --app-id <app-id> --environment-name <env>`
- Verify API endpoint is set in environment variables
- Check if API is deployed and accessible
- Review application code for correct API endpoint usage

SHOULD:
- Check CORS configuration on the API
- Verify authentication tokens are being sent correctly
- Check API Gateway or AppSync configuration if applicable

MAY:
- Test API endpoint directly with curl
- Review CloudWatch logs for API errors

## Phase 2 — Remediate

MUST:
- Set correct API endpoint in environment variables
- Fix CORS configuration to allow Amplify domain
- Ensure API is deployed and healthy

SHOULD:
- Set API URL as environment variable: `aws amplify update-branch --app-id <app-id> --branch-name <branch> --environment-variables API_URL=<url>`
- Configure CORS to allow Amplify domains

MAY:
- Use Amplify's built-in API category for automatic configuration
- Implement API health checks

## Safety Ratings

```
safety_ratings:
  - "get-backend-environment: GREEN — read-only config check"
  - "test API endpoint with curl: GREEN — read-only connectivity test"
  - "update-branch (set API_URL): YELLOW — changes runtime configuration"
  - "configure CORS: YELLOW — changes API access control"
```

## Escalation Conditions

- API integration failure affecting production application functionality
- CORS misconfiguration blocking all frontend-to-backend communication
- API authentication failures requiring backend team coordination
- API endpoint changes needed across multiple branches

## Data Sensitivity

- API endpoint URLs: MEDIUM — reveals backend service locations
- CORS configuration: LOW — access control headers
- API authentication tokens: HIGH — provide API access

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER expose API keys or tokens in environment variables without encryption
- NEVER set CORS to allow all origins (*) on production APIs

## Phase 3 — Rollback

1. If API URL was changed incorrectly, restore previous value: `aws amplify update-branch --app-id <id> --branch-name <branch> --environment-variables API_URL=<previous-url>`
2. If CORS was changed, restore previous CORS configuration on the API
3. Trigger a rebuild after restoring environment variables
4. Verify the application can communicate with the API after rollback
5. Test API authentication end-to-end

## Output Format

```yaml
root_cause: "api_integration_issue — <specific_cause>"
evidence:
  - type: api_config
    content: "<API endpoint and configuration>"
  - type: cors
    content: "<CORS settings>"
severity: MEDIUM
mitigation:
  immediate: "Fix API endpoint or CORS configuration"
  long_term: "Use Amplify API category for managed integration"
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

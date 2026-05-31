---
title: "E2 — API Access"
description: "Diagnose Wickr API access and integration issues"
status: active
severity: MEDIUM
triggers: ["API error", "API access denied", "API timeout", "integration failed"]
owner: devops-agent
objective: "Resolve Wickr API access and integration issues"
context: "Wickr API access uses Wickr-specific endpoints and authentication, not standard AWS API Gateway. Bot SDK is the primary integration method."
---

## Phase 1 — Triage

MUST:
- Verify network is active: `aws wickr get-network --network-id <id>`
- Check API credentials and token validity
- Review CloudTrail for API call errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=wickr.amazonaws.com --max-results 20`

SHOULD:
- Verify network connectivity from the API client to Wickr endpoints
- Check if API rate limits are being exceeded
- Review API client logs for specific error codes

MAY:
- Test API connectivity with a simple health check call
- Verify TLS certificate chain for API endpoints

## Phase 2 — Remediate

MUST:
- Regenerate API token if expired or revoked
- Fix network connectivity issues to Wickr API endpoints
- Ensure API client uses correct endpoint URL

SHOULD:
- Implement API retry logic with exponential backoff
- Update API client SDK to latest version
- Configure proper error handling in API integration

MAY:
- Implement API call monitoring with CloudWatch custom metrics
- Set up alerting for API error rate thresholds

## Safety Ratings

```
safety_ratings:
  - "wickr get-network: GREEN — read-only network details"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "API token regeneration: YELLOW — invalidates existing tokens"
  - "API endpoint testing: GREEN — read-only connectivity check"
```

## Escalation Conditions

- API endpoints unreachable from all clients
- API authentication fails after token regeneration
- API rate limiting affecting critical business operations
- API responses contain unexpected errors or data

## Data Sensitivity

- `aws wickr get-network`: LOW — returns network metadata
- API tokens: HIGH — authentication credentials for Wickr access
- API call logs: MEDIUM — reveal integration patterns and data flow
- API response data: MEDIUM — may contain user metadata

## Prohibited Actions

- NEVER share API tokens in logs, code repositories, or outputs
- NEVER use API tokens in client-side code
- NEVER bypass API authentication for testing
- NEVER store API tokens in plaintext configuration files

## Phase 3 — Rollback

1. If API token was regenerated, update all API clients with new token
2. If API endpoint configuration was changed, restore original settings
3. Verify all API integrations reconnect successfully
4. Confirm API-dependent workflows are functioning
5. Document API access changes and update integration documentation

## Common Issues

- symptoms: "API returns 401 Unauthorized"
  diagnosis: "API token is expired, revoked, or incorrect."
  resolution: "Regenerate API token in admin console. Update all API clients with new token."

- symptoms: "API calls timeout consistently"
  diagnosis: "Network connectivity issue between API client and Wickr endpoints."
  resolution: "Check firewall rules, proxy settings, and DNS resolution for Wickr API endpoints."

## Output Format

```yaml
root_cause: "api_access — <specific_cause>"
evidence:
  - type: api_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Restore API access"
  long_term: "Implement API monitoring and token rotation"
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

---
title: "C2 — API Asset Errors"
description: "Diagnose API asset invocation errors in Data Exchange"
status: active
severity: HIGH
triggers: ["send-api-asset error", "API asset 4xx", "API asset 5xx", "API Gateway proxy error", "API asset timeout"]
owner: devops-agent
objective: "Resolve API asset invocation issues including authentication, proxy errors, and provider backend failures"
context: "API assets proxy HTTP requests through Data Exchange to the provider's API Gateway endpoint. Errors can originate from Data Exchange (entitlement, auth) or the provider's backend (4xx, 5xx)."
---

## Phase 1 — Triage

MUST:
- Verify entitlement to the data set: `aws dataexchange get-data-set --data-set-id <dataset-id>`
- Test API asset invocation: `aws dataexchange send-api-asset --data-set-id <dataset-id> --revision-id <revision-id> --asset-id <asset-id> --method GET --path /`
- Check the HTTP status code and error message in the response

SHOULD:
- Verify the asset ID, data set ID, and revision ID are correct
- Check if the error is from Data Exchange (entitlement) or the provider's API (backend)
- Review the API asset documentation from the provider

MAY:
- Check CloudTrail for SendApiAsset events
- Verify network connectivity and DNS resolution

## Phase 2 — Remediate

MUST:
- Fix entitlement issues if Data Exchange returns 403
- Use correct HTTP method, path, headers, and body as documented by the provider
- Distinguish between Data Exchange errors and provider backend errors

SHOULD:
- Check provider's API documentation for required parameters
- Verify request body format matches the API specification

MAY:
- Contact provider if backend returns persistent 5xx errors
- Implement retry logic with exponential backoff for transient errors

## Safety Ratings

```
safety_ratings:
  - "dataexchange get-data-set: GREEN — read-only data set check"
  - "dataexchange send-api-asset (GET): GREEN — read-only API invocation"
  - "dataexchange send-api-asset (POST/PUT): YELLOW — may modify provider-side data"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- Provider API returns persistent 5xx errors indicating backend issues
- Data Exchange proxy returns errors despite valid entitlement
- API asset rate limiting prevents normal usage
- Provider API documentation is missing or incorrect

## Data Sensitivity

- `aws dataexchange send-api-asset`: HIGH — sends requests to provider API, response contains data
- `aws dataexchange get-data-set`: LOW — returns data set metadata
- `API request/response bodies`: HIGH — may contain sensitive data from the provider
- `CloudTrail SendApiAsset events`: MEDIUM — reveals API usage patterns

## Prohibited Actions

- NEVER send API requests with fabricated or manipulated authentication headers
- NEVER exceed documented rate limits intentionally
- NEVER cache or store API responses without complying with the provider's data usage terms
- NEVER expose API asset responses containing sensitive data in logs

## Phase 3 — Rollback

1. If incorrect API calls were made (POST/PUT), contact provider to reverse changes
2. If API credentials were exposed, rotate them immediately
3. Verify API access is restored after fixing entitlement or configuration
4. Document the API error pattern for future reference
5. Update client code to handle the error scenario gracefully

## Common Issues

- symptoms: "send-api-asset returns 403 — not entitled"
  diagnosis: "Subscription expired or data set ID is incorrect."
  resolution: "Verify subscription is active and use correct data set, revision, and asset IDs."

## Output Format

```yaml
root_cause: "api_asset — <specific_cause>"
evidence:
  - type: api_error
    content: "<HTTP status and error message>"
severity: HIGH
mitigation:
  immediate: "Fix API invocation"
  long_term: "Implement error handling and monitoring"
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

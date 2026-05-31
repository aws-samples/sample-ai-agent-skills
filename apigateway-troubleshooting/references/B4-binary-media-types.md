---
title: "B4 — Binary Media Type Issues"
description: "Diagnose binary media type encoding/decoding failures for images, files, and binary data"
status: active
severity: MEDIUM
triggers:
  - "binary"
  - "base64"
  - "image"
  - "file upload"
  - "file download"
  - "garbled response"
  - "isBase64Encoded"
  - "binary media type"
owner: devops-agent
objective: "Resolve binary media type issues including base64 encoding/decoding and content type configuration"
context: "API Gateway handles binary data through the binaryMediaTypes setting. When Content-Type or Accept header matches a binary media type, API Gateway base64-encodes the request body (for Lambda proxy) and expects base64-encoded response body with isBase64Encoded: true. Missing configuration causes garbled binary data or encoding errors."
---

## Phase 1 — Triage

MUST:
- Check binary media types configuration: `aws apigateway get-rest-api --rest-api-id <api-id> --query 'binaryMediaTypes'`
- Check integration content handling: `aws apigateway get-integration --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --query 'contentHandling'`
- Check integration response content handling: `aws apigateway get-integration-response --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --status-code 200 --query 'contentHandling'`
- Verify the Content-Type and Accept headers in the request match binary media types
- Test with curl: `curl -H "Accept: image/png" -o output.png https://<api-url>/path`

SHOULD:
- Check if Lambda response includes isBase64Encoded: true
- Verify the response body is actually base64-encoded
- Check if */* is in binaryMediaTypes (treats all content as binary)

MAY:
- Check API Gateway execution logs for content handling details
- Test with different Accept header values

## Phase 2 — Remediate

MUST:
- Add the required media types to binaryMediaTypes: `aws apigateway update-rest-api --rest-api-id <api-id> --patch-operations op=add,path=/binaryMediaTypes/image~1png`
- For Lambda proxy, return: `{ statusCode: 200, headers: { "Content-Type": "image/png" }, body: "<base64-string>", isBase64Encoded: true }`
- Set contentHandling to CONVERT_TO_BINARY on integration response for custom integrations
- Create a new deployment after changes

SHOULD:
- Use */* as binary media type if the API handles multiple binary formats
- Set contentHandling to CONVERT_TO_TEXT for request body if Lambda expects string input
- Test with both binary and text content types to ensure both work

MAY:
- Use multipart/form-data for file uploads with appropriate binary media type configuration
- Implement content negotiation based on Accept header

## Common Issues

- symptoms: "Garbled binary response (corrupted images/files)"
  diagnosis: "Binary media type not configured or isBase64Encoded not set in Lambda response."
  resolution: "Add media type to binaryMediaTypes, set isBase64Encoded: true, and base64-encode the body."

- symptoms: "Request body is base64-encoded when it shouldn't be"
  diagnosis: "*/* is in binaryMediaTypes, causing all content to be treated as binary."
  resolution: "Use specific media types instead of */* or set contentHandling to CONVERT_TO_TEXT."

- symptoms: "File upload fails with 400 or 500 error"
  diagnosis: "Binary media type not configured for the upload content type."
  resolution: "Add the upload content type (e.g., multipart/form-data) to binaryMediaTypes."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify configuration: YELLOW - Changes service behavior, verify in non-production first"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Certificate/TLS changes: RED - May cause downtime if misconfigured"
```

## Escalation Conditions

- API serves production traffic from external or internal consumers
- Fix requires redeploying a production stage
- Resolution involves modifying authorizer configuration
- Issue affects multiple APIs or is account-level throttling
- Custom domain name or certificate changes are required

## Data Sensitivity

MEDIUM - API keys and usage plan configurations control access to APIs. Authorizer configurations (Lambda authorizer code, Cognito pool settings) are security-sensitive. Access logs may contain request parameters, headers, and client IPs. Mapping templates may expose internal backend URLs and data transformation logic.

## Prohibited Actions

- NEVER suggest deleting a production stage - this immediately removes the API endpoint
- NEVER suggest removing all authorizers from an API that requires authentication
- NEVER recommend disabling API key requirements on usage-plan-protected endpoints
- NEVER suggest modifying throttling limits to unlimited on a production API
- NEVER recommend deleting and recreating a custom domain name - this causes DNS propagation downtime

## Phase 3 - Rollback

1. If a deployment was made, redeploy the previous working deployment to the stage: `aws apigateway create-deployment --rest-api-id <id> --stage-name <stage>`
2. If integration settings were changed, restore original integration configuration and redeploy
3. If CORS or method configuration was changed, restore original settings and redeploy the API
4. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "binary_media — <specific_cause>"
evidence:
  - type: binary_media_types
    content: "<configured binary media types>"
  - type: content_handling
    content: "<integration content handling setting>"
  - type: response_format
    content: "<Lambda response isBase64Encoded and Content-Type>"
severity: MEDIUM
mitigation:
  immediate: "Configure binary media types and fix response encoding"
  long_term: "Implement consistent binary handling across all endpoints"
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
  - "NEVER suggest removing authorizers to fix access"
  - "NEVER suggest disabling throttling"
  - "NEVER suggest open CORS for production APIs"

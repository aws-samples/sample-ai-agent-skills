---
title: "F1 — API Errors"
description: "Diagnose Control Catalog API errors"
status: active
severity: MEDIUM
triggers: ["API error", "access denied", "throttling", "API not available"]
owner: devops-agent
objective: "Resolve Control Catalog API errors"
context: "Common API errors include access denied, throttling, resource not found, and service unavailability."
---

## Phase 1 — Triage

MUST:
- Check error message and HTTP status code
- Verify IAM permissions for controlcatalog:* actions
- Check if API is available in the region

SHOULD:
- Check for throttling (429 errors) and implement backoff
- Verify AWS CLI version supports controlcatalog

MAY:
- Review CloudTrail for API call details

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for AccessDeniedException
- Implement exponential backoff for throttling
- Update CLI for UnrecognizedClientException

SHOULD:
- Handle pagination properly
- Cache results to reduce API calls

MAY:
- Use SDK with built-in retry logic

## Common Issues

- symptoms: "UnrecognizedClientException for controlcatalog"
  diagnosis: "AWS CLI version doesn't support controlcatalog service."
  resolution: "Update AWS CLI to latest version: pip install --upgrade awscli."

- symptoms: "ThrottlingException on list-controls"
  diagnosis: "Too many API calls in short period."
  resolution: "Implement exponential backoff. Cache results."

## Output Format

```yaml
root_cause: "api_errors — <specific_cause>"
evidence:
  - type: api_error
    content: "<error code and message>"
severity: MEDIUM
mitigation:
  immediate: "Fix specific API error"
  long_term: "Implement proper error handling and caching"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — API errors involve public catalog operations |
| Blast Radius | MEDIUM — API failures block compliance tooling and assessments |
| Rollback Complexity | LOW — Fix configuration and retry |
| Compliance Scope | MEDIUM — Persistent API failures can delay compliance assessments |

## Escalation Conditions

- Escalate to operations if API errors block automated compliance scanning
- Escalate to AWS Support if throttling persists despite implementing backoff
- Escalate to compliance team if API unavailability delays audit timelines

## Data Sensitivity

- **API error messages** may contain account identifiers — handle per standard practices
- **CloudTrail events** for API calls are standard audit records

## PROHIBITED Actions

- Do NOT retry API calls without implementing exponential backoff
- Do NOT grant overly broad IAM permissions to fix access denied errors
- Do NOT ignore persistent API errors — they may indicate configuration drift

## Phase 3 — Rollback

- Revert any IAM permission changes made during troubleshooting
- Restore previous CLI version if upgrade caused issues
- Re-enable caching if it was disabled during troubleshooting

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

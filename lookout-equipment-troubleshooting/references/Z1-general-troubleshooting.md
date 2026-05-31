---
title: "Z1 — General Troubleshooting"
description: "General troubleshooting for Amazon Lookout for Equipment issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["lookout for equipment error", "general issue", "unknown error", "service error"]
owner: devops-agent
objective: "Provide systematic approach to diagnosing unclassified Lookout for Equipment issues"
context: "Catch-all runbook for issues that do not match specific runbook triggers. Follows a systematic elimination approach."
---

## Phase 1 — Triage

MUST:
- Identify the specific API call or operation that is failing
- Check service health: `aws health describe-events --filter '{"services":["LOOKOUTEQUIPMENT"]}'`
- Verify AWS CLI version supports lookoutequipment commands: `aws lookoutequipment help`
- Check caller identity and region: `aws sts get-caller-identity` and verify region setting

SHOULD:
- Review CloudTrail for recent Lookout for Equipment events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=lookoutequipment.amazonaws.com --max-results 20`
- Check service quotas: `aws service-quotas list-service-quotas --service-code lookoutequipment`
- Verify the region supports Lookout for Equipment

MAY:
- Check for recent AWS service announcements or known issues
- Review AWS Personal Health Dashboard for account-specific notifications
- Check if the issue correlates with recent infrastructure changes

## Phase 2 — Remediate

MUST:
- Route to the appropriate specific runbook based on triage findings
- If API error: check IAM permissions (see E2-iam-permissions)
- If data error: check data format and S3 access (see A2, E1)
- If model error: check training data and configuration (see B1, B2)

SHOULD:
- Document the error message, timestamp, and request ID for AWS Support
- Check if the issue is transient by retrying after a few minutes
- Verify no recent changes to IAM policies, S3 configurations, or VPC settings

MAY:
- Open an AWS Support case with the error details and request ID
- Check AWS re:Post for similar issues reported by other users

## Common Issues

- symptoms: "UnrecognizedClientException or endpoint errors"
  diagnosis: "The AWS CLI or SDK does not support Lookout for Equipment, or the region does not have the service."
  resolution: "Update AWS CLI to latest version. Verify the service is available in the target region. Check endpoint URL configuration."

- symptoms: "ThrottlingException on API calls"
  diagnosis: "API rate limits exceeded due to too many concurrent requests."
  resolution: "Implement exponential backoff and retry logic. Reduce API call frequency. Check service quotas for rate limits."

- symptoms: "InternalServerException on any operation"
  diagnosis: "Transient service-side error."
  resolution: "Retry after 30 seconds with exponential backoff. If persistent, check AWS Health Dashboard and open a support case."

## Safety Ratings

```
safety_ratings:
  - "health describe-events: GREEN — read-only service health check"
  - "cloudtrail lookup-events: GREEN — read-only audit log query"
  - "service-quotas list-service-quotas: GREEN — read-only quota check"
  - "sts get-caller-identity: GREEN — read-only identity verification"
```

## Escalation Conditions

- Internal service errors persist for more than 1 hour
- The issue affects multiple Lookout for Equipment resources simultaneously
- Service quotas need to be increased and the request is urgent
- The issue cannot be classified into any specific runbook category

## Data Sensitivity

- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principals and API actions
- `aws health describe-events`: LOW — returns service health status
- Error messages and request IDs: LOW — technical debugging information
- Service quota values: LOW — account configuration metadata

## Prohibited Actions

- NEVER assume a service error is permanent without retrying with backoff
- NEVER share AWS request IDs or CloudTrail logs externally without redacting account information
- NEVER attempt to work around service quotas by creating multiple accounts
- NEVER ignore persistent errors — they may indicate a deeper configuration issue

## Phase 3 — Rollback

1. If troubleshooting changes caused new issues, revert all changes in reverse order
2. Document all changes made during troubleshooting for audit trail
3. If the issue requires AWS Support, provide the request ID, timestamp, region, and error message
4. Restore any modified IAM policies, S3 configurations, or resource settings to their pre-troubleshooting state
5. Notify the team of the issue status and any temporary workarounds in place

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: error_details
    content: "<error message and context>"
severity: MEDIUM
mitigation:
  immediate: "Apply specific runbook or retry with backoff"
  long_term: "Implement monitoring and alerting for Lookout for Equipment operations"
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

---
title: "A1 — Flow Creation Failures"
description: "Diagnose Amazon AppFlow flow creation failures"
status: active
severity: HIGH
triggers: ["cannot create flow", "flow creation failed", "CreateFlow error"]
owner: devops-agent
objective: "Resolve flow creation issues including IAM, connector, and configuration errors"
context: "Flows require a valid connector profile, IAM permissions, and compatible source-destination configuration."
---

## Phase 1 — Triage

MUST:
- Check existing flows: `aws appflow list-flows`
- Verify IAM permissions for appflow:CreateFlow
- Verify connector profile exists: `aws appflow list-connector-profiles`

SHOULD:
- Check service quotas: `aws service-quotas get-service-quota --service-code appflow --quota-code L-6030F361`
- Verify source and destination connector compatibility

MAY:
- Review CloudTrail for CreateFlow errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=appflow.amazonaws.com --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for appflow:CreateFlow, appflow:TagResource
- Ensure connector profile is valid and in the same region
- Verify source-destination connector pair is supported

SHOULD:
- Request quota increase if at the 100-flow limit
- Validate field mappings before creation

MAY:
- Use CloudFormation or Terraform for repeatable flow creation

## Safety Ratings

```
safety_ratings:
  - "list-flows: GREEN — read-only enumeration of existing flows"
  - "list-connector-profiles: GREEN — read-only enumeration of connector profiles"
  - "CreateFlow: YELLOW — creates a new flow resource, non-destructive"
  - "service-quotas get-service-quota: GREEN — read-only quota check"
```

## Escalation Conditions

- Flow creation fails with internal service errors after fixing IAM and configuration
- Connector profile validation passes but flow creation still fails
- Service quota increase request is denied
- Flow creation blocked by SCP or permission boundary

## Data Sensitivity

- `aws appflow list-flows`: LOW — returns flow metadata only
- `aws appflow list-connector-profiles`: MEDIUM — reveals SaaS integration details
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER create flows with overly broad IAM permissions as a workaround
- NEVER delete existing connector profiles to troubleshoot creation issues
- NEVER hardcode OAuth credentials in flow configurations
- NEVER bypass encryption requirements to simplify flow creation

## Phase 3 — Rollback

1. If a flow was created with incorrect configuration, delete it: `aws appflow delete-flow --flow-name <name> --force-delete`
2. If a connector profile was created for testing, remove it: `aws appflow delete-connector-profile --connector-profile-name <name>`
3. Verify no scheduled executions were triggered by the misconfigured flow
4. Check S3 destination for any partial data written during test executions
5. Document the rollback action and correct configuration for retry

## Common Issues

- symptoms: "CreateFlow returns ValidationException for field mappings"
  diagnosis: "Source schema fields do not match the configured field mappings."
  resolution: "Re-fetch source schema and update field mappings to match current source fields."

## Output Format

```yaml
root_cause: "flow_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation issue"
  long_term: "Use IaC for flow deployment"
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

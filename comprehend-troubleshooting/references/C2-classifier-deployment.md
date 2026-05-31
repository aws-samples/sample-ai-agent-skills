---
title: "C2 — Custom Classifier Deployment Issues"
description: "Diagnose classifier endpoint creation and management problems"
status: active
severity: HIGH
triggers:
  - "endpoint creation failed"
  - "CreateEndpoint error"
  - "endpoint not ready"
owner: devops-agent
objective: "Identify and resolve classifier deployment issues"
context: "Custom classifiers require endpoints for real-time inference. Endpoints are billed per inference unit while active. Endpoint creation takes several minutes."
---

## Phase 1 — Triage

MUST:
- List endpoints: `aws comprehend list-endpoints`
- Check endpoint status: `aws comprehend describe-endpoint --endpoint-arn <arn>`
- Verify classifier is TRAINED status
- Check IAM permissions for CreateEndpoint

SHOULD:
- Verify inference unit count is appropriate
- Check endpoint model ARN matches intended classifier
- Monitor endpoint utilization

MAY:
- Check for endpoint quota limits
- Review endpoint update history

## Phase 2 — Remediate

MUST:
- Ensure classifier is TRAINED before creating endpoint
- Create endpoint with appropriate inference units
- Wait for IN_SERVICE status before inference

SHOULD:
- Scale inference units based on throughput needs
- Delete unused endpoints to stop billing

MAY:
- Implement endpoint auto-scaling

## Common Issues

- symptoms: "CreateEndpoint fails"
  diagnosis: "Classifier not in TRAINED status or quota exceeded."
  resolution: "Wait for training completion. Check endpoint quota."

- symptoms: "Endpoint stuck in CREATING status"
  diagnosis: "Endpoint creation takes several minutes."
  resolution: "Wait for IN_SERVICE status. Check for errors after 15 minutes."

## Output Format

```yaml
root_cause: "classifier_deployment — <specific_cause>"
evidence:
  - type: endpoint_status
    content: "<endpoint details>"
  - type: classifier_status
    content: "<classifier status>"
severity: HIGH
mitigation:
  immediate: "Fix classifier status or endpoint configuration"
  long_term: "Implement endpoint lifecycle management"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Endpoints process documents that may contain sensitive content |
| Blast Radius | MEDIUM — Endpoint creation incurs continuous costs; misconfigured endpoints affect classification |
| Rollback Complexity | LOW — Endpoints can be deleted immediately to stop costs |
| Compliance Scope | MEDIUM — Production endpoints must serve validated models |

## Escalation Conditions

- Escalate to management if unused endpoints are running and incurring costs
- Escalate to operations if endpoint creation repeatedly fails blocking production deployment
- Escalate to AWS Support if endpoints get stuck in CREATING state beyond 30 minutes

## Data Sensitivity

- **Inference requests** may contain sensitive documents — ensure TLS in transit
- **Endpoint ARN** should be treated as a resource identifier with controlled access
- **Inference unit scaling** affects cost — monitor utilization to avoid waste

## PROHIBITED Actions

- Do NOT leave test or unused endpoints running in production accounts
- Do NOT deploy endpoints for unvalidated classifier models to production
- Do NOT scale inference units beyond approved budget without authorization
- Do NOT share endpoint ARNs with unauthorized parties

## Phase 3 — Rollback

- Delete the problematic endpoint: `aws comprehend delete-endpoint --endpoint-arn <arn>`
- Recreate endpoint pointing to the previous validated classifier model
- If endpoint was serving production traffic, update application to use the replacement endpoint ARN
- Monitor replacement endpoint for IN_SERVICE status before routing traffic

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

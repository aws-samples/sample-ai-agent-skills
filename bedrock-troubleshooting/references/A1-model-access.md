---
title: "A1 — Model Access Issues"
description: "Diagnose model access and availability problems in Bedrock"
status: active
severity: HIGH
triggers:
  - "model access denied"
  - "model not available"
  - "AccessDeniedException invoke model"
  - "model access request"
owner: devops-agent
objective: "Identify and resolve model access issues in Bedrock"
context: "Each foundation model requires explicit access approval per region. Access is requested through the Bedrock console. Some models require additional terms. Model availability varies by region. Without approval, InvokeModel returns AccessDeniedException."
---

## Phase 1 — Triage

MUST:
- Check model access status in Bedrock console (Model access page)
- Verify the model ID is correct: `aws bedrock list-foundation-models --query 'modelSummaries[?modelId==\`<model-id>\`]'`
- Check the region — model access is per-region
- Verify IAM permissions include bedrock:InvokeModel

SHOULD:
- Check if the model is available in the target region
- Verify the model lifecycle status (ACTIVE vs LEGACY)
- Check CloudTrail for AccessDeniedException events
- Verify no SCPs block Bedrock operations

MAY:
- Check for model deprecation notices
- Verify account-level Bedrock access

## Phase 2 — Remediate

MUST:
- Request model access through the Bedrock console
- Wait for access approval (usually immediate for most models)
- Verify the correct model ID format

SHOULD:
- Request access in all needed regions
- Document which models are approved per region
- Use the latest model version when available

MAY:
- Set up model access monitoring
- Create model access request automation

## Common Issues

- symptoms: "AccessDeniedException when invoking model"
  diagnosis: "Model access not approved or wrong region."
  resolution: "Request model access in the Bedrock console for the correct region."

- symptoms: "Model ID not found"
  diagnosis: "Incorrect model ID format or model not available in region."
  resolution: "Use aws bedrock list-foundation-models to find correct model IDs."

## Output Format

```yaml
root_cause: "model_access — <specific_cause>"
evidence:
  - type: model_status
    content: "<model access and availability status>"
  - type: iam_permissions
    content: "<IAM policy evaluation>"
severity: HIGH
mitigation:
  immediate: "Request model access and fix permissions"
  long_term: "Document model access requirements per region"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Request model access through the Bedrock console | GREEN |
| Wait for access approval | GREEN |
| Verify the correct model ID format | GREEN |
| Request access in all needed regions | GREEN |
| Document which models are approved per region | GREEN |
| Use the latest model version when available | YELLOW |
| Set up model access monitoring | GREEN |
| Create model access request automation | YELLOW |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Model access status and IAM policy details may reveal security posture

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If model access was granted in error, revoke access via the Bedrock console Model Access page
2. If IAM permissions were broadened, revert the IAM policy to the previous version using IAM policy versioning
3. If the wrong model version was selected, update the application to reference the previous model ID
4. Verify rollback by confirming `aws bedrock list-foundation-models` output matches expected state

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
  - "NEVER suggest removing guardrails from production models"
  - "NEVER suggest disabling content filtering"
  - "NEVER suggest overly broad model access permissions"

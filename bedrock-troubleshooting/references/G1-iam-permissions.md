---
title: "G1 — IAM Permission Issues"
description: "Diagnose IAM permission problems for Bedrock operations"
status: active
severity: HIGH
triggers:
  - "access denied Bedrock"
  - "IAM permission error"
  - "unauthorized Bedrock"
  - "Bedrock permission"
owner: devops-agent
objective: "Identify and resolve IAM permission issues for Bedrock"
context: "Bedrock operations require specific IAM permissions. InvokeModel needs bedrock:InvokeModel. Agent operations need bedrock-agent:*. Knowledge base operations need bedrock-agent:* plus S3 and vector store permissions. Model access approval is separate from IAM permissions."
---

## Phase 1 — Triage

MUST:
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names bedrock:InvokeModel bedrock:ListFoundationModels`
- Verify the specific action being denied
- Check for SCPs blocking Bedrock operations
- Distinguish between IAM denial and model access denial

SHOULD:
- Check resource-level permissions (specific model ARN)
- Verify condition keys in IAM policies
- Check for permission boundaries
- Review CloudTrail for denied requests

MAY:
- Check for cross-account access requirements
- Verify VPC endpoint policies

## Phase 2 — Remediate

MUST:
- Add required Bedrock IAM permissions
- For invocation: bedrock:InvokeModel, bedrock:InvokeModelWithResponseStream
- For agents: bedrock-agent:InvokeAgent
- For KB: bedrock-agent:Retrieve, bedrock-agent:RetrieveAndGenerate

SHOULD:
- Use least-privilege permissions
- Scope permissions to specific model ARNs if possible
- Document required permissions per use case

MAY:
- Create IAM policy templates for Bedrock
- Implement permission monitoring

## Common Issues

- symptoms: "AccessDeniedException for InvokeModel"
  diagnosis: "IAM policy missing bedrock:InvokeModel OR model access not approved."
  resolution: "Add bedrock:InvokeModel to IAM policy AND request model access in console."

- symptoms: "Access denied for agent operations"
  diagnosis: "Missing bedrock-agent:* permissions."
  resolution: "Add bedrock-agent:InvokeAgent and related permissions."

## Output Format

```yaml
root_cause: "iam_permissions — <specific_cause>"
evidence:
  - type: iam_evaluation
    content: "<IAM policy simulation results>"
  - type: denied_action
    content: "<specific denied action>"
severity: HIGH
mitigation:
  immediate: "Add required IAM permissions"
  long_term: "Create standardized IAM policies for Bedrock use cases"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Add required Bedrock IAM permissions | YELLOW |
| For invocation: bedrock:InvokeModel, bedrock:InvokeModelWithResponseStream | YELLOW |
| For agents: bedrock-agent:InvokeAgent | YELLOW |
| For KB: bedrock-agent:Retrieve, bedrock-agent:RetrieveAndGenerate | YELLOW |
| Use least-privilege permissions | GREEN |
| Scope permissions to specific model ARNs if possible | GREEN |
| Document required permissions per use case | GREEN |
| Create IAM policy templates for Bedrock | GREEN |
| Implement permission monitoring | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- IAM policies and permission configurations reveal the security boundary of Bedrock access

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If IAM permissions were broadened, revert the IAM policy to the previous version using policy versioning
2. If new IAM policies were attached, detach them from the role/user
3. If SCPs were modified, restore the previous SCP version in AWS Organizations
4. Verify rollback by running `aws iam simulate-principal-policy` to confirm permissions match expected state

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

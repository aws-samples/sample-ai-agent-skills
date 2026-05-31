---
title: "C2 — Action Group Errors"
description: "Diagnose Bedrock agent action group failures"
status: active
severity: HIGH
triggers:
  - "action group error"
  - "Lambda invocation failed"
  - "action group not working"
  - "API schema error"
owner: devops-agent
objective: "Identify and resolve agent action group errors"
context: "Action groups define tools the agent can use. They connect to Lambda functions or API endpoints via OpenAPI schemas. Failures stem from Lambda errors, schema mismatches, permission issues, or incorrect response formats."
---

## Phase 1 — Triage

MUST:
- Check action group configuration: `aws bedrock-agent get-agent-action-group --agent-id <id> --agent-version DRAFT --action-group-id <ag-id>`
- Verify Lambda function exists and is accessible
- Check Lambda function logs in CloudWatch
- Verify OpenAPI schema is valid

SHOULD:
- Test Lambda function independently
- Check agent execution role has lambda:InvokeFunction permission
- Verify Lambda response format matches Bedrock expectations
- Check for Lambda timeout issues

MAY:
- Review OpenAPI schema for completeness
- Check Lambda concurrency limits

## Phase 2 — Remediate

MUST:
- Fix Lambda function errors
- Ensure agent role has lambda:InvokeFunction permission
- Verify Lambda response format (must include specific fields)

SHOULD:
- Test Lambda independently before connecting to agent
- Use descriptive action group and API descriptions for better agent routing
- Set appropriate Lambda timeout (agent has its own timeout)

MAY:
- Create Lambda function templates for action groups
- Implement action group monitoring

## Common Issues

- symptoms: "Agent fails to invoke action group"
  diagnosis: "Agent role lacks lambda:InvokeFunction or Lambda ARN incorrect."
  resolution: "Add lambda:InvokeFunction to agent role. Verify Lambda ARN."

- symptoms: "Action group invoked but returns error"
  diagnosis: "Lambda function error or response format incorrect."
  resolution: "Check Lambda logs. Ensure response matches Bedrock action group format."

## Output Format

```yaml
root_cause: "action_group_error — <specific_cause>"
evidence:
  - type: action_group_config
    content: "<action group configuration>"
  - type: lambda_logs
    content: "<Lambda execution logs>"
severity: HIGH
mitigation:
  immediate: "Fix Lambda function or permissions"
  long_term: "Create validated action group templates"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix Lambda function errors | YELLOW |
| Ensure agent role has lambda:InvokeFunction permission | YELLOW |
| Verify Lambda response format | GREEN |
| Test Lambda independently before connecting to agent | GREEN |
| Use descriptive action group and API descriptions | GREEN |
| Set appropriate Lambda timeout | GREEN |
| Create Lambda function templates for action groups | GREEN |
| Implement action group monitoring | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Lambda function logs may contain sensitive request/response data from agent interactions

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If Lambda function was modified, redeploy the previous version: `aws lambda update-function-code --function-name <name> --s3-bucket <bucket> --s3-key <previous-version-key>`
2. If IAM permissions were broadened, revert the agent role policy to the previous version
3. If the OpenAPI schema was changed, restore the previous schema and update the action group
4. Run `aws bedrock-agent prepare-agent --agent-id <id>` after reverting changes
5. Verify rollback by testing the agent action group with a known input

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

---
title: "H1 — Lambda Integration Issues"
description: "Diagnose Lambda integration problems with Bedrock"
status: active
severity: MEDIUM
triggers:
  - "Lambda Bedrock timeout"
  - "Lambda invocation Bedrock"
  - "Lambda agent integration"
  - "Lambda Bedrock error"
owner: devops-agent
objective: "Identify and resolve Lambda integration issues with Bedrock"
context: "Lambda functions commonly invoke Bedrock models, serve as agent action groups, or process KB results. Issues include timeout (Bedrock calls can be slow), IAM permissions, VPC configuration, and response handling. Lambda timeout must account for Bedrock response time."
---

## Phase 1 — Triage

MUST:
- Check Lambda function configuration: `aws lambda get-function-configuration --function-name <name>`
- Verify Lambda execution role has Bedrock permissions
- Check Lambda timeout setting (must be > Bedrock response time)
- Review Lambda CloudWatch logs for errors

SHOULD:
- Check if Lambda is in a VPC (needs VPC endpoint for Bedrock)
- Verify Lambda memory allocation (SDK operations need memory)
- Check for cold start impact on total latency
- Review error handling in Lambda code

MAY:
- Check Lambda concurrency limits
- Review Lambda layers for SDK version

## Phase 2 — Remediate

MUST:
- Set Lambda timeout to at least 30-60 seconds for Bedrock calls
- Add bedrock:InvokeModel to Lambda execution role
- If in VPC, create Bedrock VPC endpoints

SHOULD:
- Increase Lambda memory for faster SDK operations
- Implement proper error handling and retry logic
- Use streaming for long responses to avoid timeout

MAY:
- Use provisioned concurrency to reduce cold starts
- Implement Lambda response caching

## Common Issues

- symptoms: "Lambda times out when calling Bedrock"
  diagnosis: "Lambda timeout too short for Bedrock response time."
  resolution: "Increase Lambda timeout to 60+ seconds. Use streaming for long responses."

- symptoms: "Lambda in VPC cannot reach Bedrock"
  diagnosis: "No VPC endpoint for Bedrock in the Lambda's VPC."
  resolution: "Create bedrock-runtime VPC interface endpoint."

## Output Format

```yaml
root_cause: "lambda_integration — <specific_cause>"
evidence:
  - type: lambda_config
    content: "<Lambda configuration>"
  - type: execution_logs
    content: "<Lambda execution logs>"
severity: MEDIUM
mitigation:
  immediate: "Fix Lambda timeout, permissions, or VPC configuration"
  long_term: "Create Lambda templates for Bedrock integration"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Set Lambda timeout to at least 30-60 seconds for Bedrock calls | GREEN |
| Add bedrock:InvokeModel to Lambda execution role | YELLOW |
| If in VPC, create Bedrock VPC endpoints | YELLOW |
| Increase Lambda memory for faster SDK operations | GREEN |
| Implement proper error handling and retry logic | GREEN |
| Use streaming for long responses to avoid timeout | GREEN |
| Use provisioned concurrency to reduce cold starts | YELLOW |
| Implement Lambda response caching | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Lambda function logs and environment variables may contain API keys, model parameters, or sensitive data

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If Lambda timeout was changed, restore the previous timeout value: `aws lambda update-function-configuration --function-name <name> --timeout <previous-value>`
2. If Lambda execution role was modified, revert the IAM policy to the previous version
3. If VPC endpoints were created for Lambda, delete them if no longer needed
4. If Lambda memory was increased, restore the previous memory setting
5. Verify rollback by invoking the Lambda function and confirming expected behavior

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

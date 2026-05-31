---
title: "G1 — Custom Config Rules"
description: "Diagnose and resolve custom AWS Config rule issues in LZA"
status: active
severity: MEDIUM
triggers: ["custom Config rule error", "Config rule not evaluating", "custom rule deployment failed", "compliance check error"]
owner: devops-agent
objective: "Ensure custom AWS Config rules are correctly deployed and evaluating compliance"
context: "LZA supports deploying custom AWS Config rules via security-config.yaml. Custom rules use Lambda functions for evaluation. Issues include rule deployment failures, Lambda function errors, and incorrect evaluation logic."
---

## Phase 1 — Triage

MUST:
- List Config rules: `aws configservice describe-config-rules --query 'ConfigRules[?contains(ConfigRuleName,`Accelerator`)].{Name:ConfigRuleName,State:ConfigRuleState}'`
- Check rule compliance: `aws configservice describe-compliance-by-config-rule --config-rule-names <rule-name>`
- Verify Lambda function for custom rule: `aws lambda get-function --function-name <function-name>`

SHOULD:
- Check Lambda function logs for evaluation errors: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch_ms>`
- Verify Config rule trigger type (periodic vs configuration change)
- Check Config recorder status: `aws configservice describe-configuration-recorder-status`

MAY:
- Test the Lambda function with a sample configuration item
- Check for Config rule evaluation throttling
- Review the custom rule Lambda function code

## Phase 2 — Remediate

MUST:
- Fix Lambda function errors identified in CloudWatch Logs
- Ensure Config recorder is running: `aws configservice start-configuration-recorder --configuration-recorder-name default`
- Verify Lambda function has permissions to be invoked by Config

SHOULD:
- Add resource-based policy for Config to invoke Lambda: `aws lambda add-permission --function-name <name> --statement-id config-invoke --action lambda:InvokeFunction --principal config.amazonaws.com`
- Fix evaluation logic in the Lambda function
- Verify the rule scope matches the intended resource types

MAY:
- Implement unit tests for custom rule Lambda functions
- Set up Config rule remediation actions

## Safety Ratings

```
safety_ratings:
  - "configservice describe-config-rules: GREEN — read-only rule listing"
  - "configservice describe-compliance: GREEN — read-only compliance check"
  - "lambda get-function: GREEN — read-only function details"
  - "Config rule deployment: YELLOW — creates evaluation rules across accounts"
```

## Escalation Conditions

- Custom Config rules produce false compliance results affecting audit reports
- Lambda function consistently fails evaluation causing NON_COMPLIANT status
- Config rule quota limits reached in target accounts
- Custom rules conflict with LZA-managed Config rules

## Data Sensitivity

- Config rule compliance data: MEDIUM — reveals resource compliance status
- Lambda function code: MEDIUM — reveals evaluation logic
- Lambda function logs: MEDIUM — may contain resource configuration details
- Config rule scope: LOW — resource type and scope information

## Prohibited Actions

- NEVER delete Config rules that enforce compliance requirements without approval
- NEVER modify LZA-managed Config rules outside of LZA configuration
- NEVER disable the Config recorder to fix rule evaluation issues
- NEVER deploy custom rules that override LZA-managed compliance checks

## Phase 3 — Rollback

1. If custom rules were deployed incorrectly, remove them from security-config.yaml and re-run
2. If Lambda function was modified, deploy the previous version
3. If Config recorder was stopped, restart it immediately
4. Document all Config rule changes and their compliance impact
5. Verify compliance status is accurate after changes

## Common Issues

- symptoms: "Custom Config rule shows INSUFFICIENT_DATA"
  diagnosis: "Config recorder is not recording the resource type, or Lambda function is not being invoked."
  resolution: "Check Config recorder scope. Verify Lambda permission for Config invocation."

- symptoms: "Custom rule evaluates all resources as NON_COMPLIANT"
  diagnosis: "Lambda function evaluation logic has a bug or returns incorrect compliance status."
  resolution: "Review Lambda function code and logs. Test with a known-compliant resource."

## Output Format

```yaml
root_cause: "custom_config_rule — <specific_cause>"
evidence:
  - type: config_rule_error
    content: "<rule status, Lambda logs, and compliance details>"
severity: MEDIUM
mitigation:
  immediate: "Fix Config rule or Lambda function"
  long_term: "Implement Config rule testing and validation"
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

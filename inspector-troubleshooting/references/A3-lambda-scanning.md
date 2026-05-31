---
title: "A3 — Lambda Scanning"
description: "Diagnose Lambda function scanning issues with Inspector"
status: active
severity: MEDIUM
triggers:
  - "Lambda not scanned"
  - "Lambda scanning failed"
  - "Lambda findings missing"
  - "code scanning not working"
owner: devops-agent
objective: "Ensure Lambda functions are being scanned by Inspector"
context: "Inspector supports two Lambda scan types: standard scanning (runtime/layer CVEs) and code scanning (code vulnerabilities). Both must be enabled separately. Code scanning is available in fewer regions. Functions using unsupported runtimes may not be scanned."
---

## Phase 1 — Triage

MUST:
- Check Lambda scanning status: `aws inspector2 batch-get-account-status --query 'Accounts[*].{Lambda:ResourceState.Lambda.Status,LambdaCode:ResourceState.LambdaCode.Status}'`
- Check Lambda coverage: `aws inspector2 list-coverage --filter-criteria '{"resourceType":[{"comparison":"EQUALS","value":"AWS_LAMBDA_FUNCTION"}]}'`
- Check function runtime: `aws lambda get-function --function-name <function-name> --query 'Configuration.Runtime'`

SHOULD:
- List Lambda findings: `aws inspector2 list-findings --filter-criteria '{"resourceType":[{"comparison":"EQUALS","value":"AWS_LAMBDA_FUNCTION"}]}' --max-results 10`
- Check if the runtime is supported by Inspector
- Verify code scanning is available in the current region

MAY:
- Check function layers for known vulnerabilities
- Review function deployment history

## Phase 2 — Remediate

MUST:
- Enable Lambda standard scanning: `aws inspector2 enable --resource-types LAMBDA`
- Enable Lambda code scanning: `aws inspector2 enable --resource-types LAMBDA_CODE`
- Verify functions use supported runtimes

SHOULD:
- Update functions to supported runtimes if using deprecated ones
- Check coverage after enabling to verify functions are scanned

MAY:
- Exclude test/development functions using tag-based rules
- Set up EventBridge rules for Lambda finding notifications

## Common Issues

- symptoms: "Lambda scanning enabled but no functions covered"
  diagnosis: "Functions use unsupported runtimes."
  resolution: "Update functions to supported runtimes (Python 3.8+, Node.js 16+, Java 11+, etc.)."

- symptoms: "Standard scanning works but code scanning doesn't"
  diagnosis: "Code scanning not enabled or not available in the region."
  resolution: "Enable code scanning separately. Check regional availability."

- symptoms: "Function was scanned but findings disappeared"
  diagnosis: "Function was deleted or updated, closing previous findings."
  resolution: "Check function deployment history. Findings close when the vulnerability is resolved."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable Lambda standard scanning | GREEN | Enables vulnerability detection |
| Enable Lambda code scanning | GREEN | Enables code vulnerability detection |
| Update functions to supported runtimes | YELLOW | Runtime changes may affect function behavior |

## Escalation Conditions

- Production Lambda functions not being scanned — escalate to security operations
- Runtime updates required for scanning support — escalate to application team
- Code scanning findings in production functions — escalate to development team
- Scanning configuration changes affecting organization — escalate to delegated admin

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws inspector2 batch-get-account-status` | MEDIUM | Exposes scanning status |
| `aws inspector2 list-coverage` (Lambda) | MEDIUM | Exposes function coverage |
| `aws lambda get-function` | MEDIUM | Exposes function configuration |
| `aws inspector2 list-findings` (Lambda) | MEDIUM | Exposes vulnerability findings |

## Prohibited Actions

- **NEVER** suggest disabling Inspector scanning as a troubleshooting step
- **NEVER** suggest suppressing all findings without individual review
- **NEVER** update Lambda runtimes in production without testing
- **NEVER** ignore code scanning findings without security review

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled Lambda scanning | Disable: `aws inspector2 disable --resource-types LAMBDA` |
| Enabled Lambda code scanning | Disable: `aws inspector2 disable --resource-types LAMBDA_CODE` |
| Updated function runtime | Revert to previous runtime version |

## Output Format

```yaml
root_cause: "lambda_scanning — <specific_cause>"
evidence:
  - type: scanning_status
    content: "<Lambda scanning status>"
  - type: coverage
    content: "<Lambda function coverage>"
severity: MEDIUM
mitigation:
  immediate: "Enable Lambda scanning and verify coverage"
  long_term: "Maintain supported runtimes across all functions"
```

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
  - "NEVER suggest disabling scanning to reduce findings"
  - "NEVER suggest ignoring critical vulnerabilities"
  - "NEVER suggest removing SSM agent to avoid scans"

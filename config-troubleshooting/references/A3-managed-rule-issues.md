---
title: "A3 — Managed Rule Issues"
description: "Diagnose issues with AWS Config managed rules"
status: active
severity: MEDIUM
triggers:
  - "managed rule not working"
  - "managed rule wrong result"
  - "managed rule parameters"
  - "AWS managed rule"
  - "predefined rule issue"
owner: devops-agent
objective: "Identify and resolve issues with AWS Config managed rules including parameter misconfiguration and unexpected compliance results"
context: "Managed rules are predefined by AWS and evaluate specific resource configurations. Issues typically arise from incorrect parameters, unsupported resource types, missing recorder coverage, or misunderstanding what the rule actually checks."
---

## Phase 1 — Triage

MUST:
- Describe the managed rule: `aws configservice describe-config-rules --config-rule-names <rule-name>`
- Check rule evaluation status: `aws configservice describe-config-rule-evaluation-status --config-rule-names <rule-name>`
- Get compliance details: `aws configservice get-compliance-details-by-config-rule --config-rule-name <rule-name> --compliance-types NON_COMPLIANT`
- Verify recorder is recording the rule's target resource type: `aws configservice describe-configuration-recorders`
- Check the rule's input parameters match expected values

SHOULD:
- Get the resource configuration for a non-compliant resource: `aws configservice get-resource-config-history --resource-type <type> --resource-id <id> --limit 1`
- Verify the resource exists and is in the expected state
- Check if the rule source identifier matches the intended managed rule

MAY:
- Compare rule parameters against AWS documentation for the specific managed rule
- Check for recently added or modified resources that should trigger evaluation

## Phase 2 — Remediate

MUST:
- Correct any misconfigured rule parameters
- Ensure the recorder covers the resource types the rule evaluates
- Force re-evaluation after parameter changes: `aws configservice start-config-rules-evaluation --config-rule-names <rule-name>`

SHOULD:
- Document the correct parameter values for the managed rule
- Verify compliance results match expected behavior after re-evaluation

MAY:
- Create a conformance pack to manage related rules together
- Set up compliance change notifications via SNS

## Common Issues

- symptoms: "Managed rule shows all resources as COMPLIANT when some should be NON_COMPLIANT"
  diagnosis: "Rule parameters are too permissive or the rule checks different attributes than expected."
  resolution: "Review the rule's documentation for exact evaluation logic and adjust parameters."

- symptoms: "Managed rule shows NOT_APPLICABLE for all resources"
  diagnosis: "The resource type in the rule scope is not being recorded or does not exist in the account."
  resolution: "Verify the recorder includes the resource type and resources of that type exist."

- symptoms: "Rule parameter validation error on creation"
  diagnosis: "The parameter values do not match the expected format for the managed rule."
  resolution: "Check AWS documentation for the specific managed rule's parameter format and valid values."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Correct rule parameters | YELLOW | Rule modification; reversible |
| Force re-evaluation | GREEN | Triggers evaluation; no destructive change |
| Ensure recorder coverage | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Recorder configuration changes
- Remediation action deployment
- Rule parameter changes affecting compliance results

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-config-rules` | LOW | Rule configuration |
| `get-compliance-details-by-config-rule` | MEDIUM | Resource compliance details |
| `get-resource-config-history` | MEDIUM | Resource configuration history |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Rule parameter change | Revert parameters via `put-config-rule` |
| Recorder coverage change | Revert recorder configuration |

## Output Format

```yaml
root_cause: "managed_rule_issue — <specific_cause>"
evidence:
  - type: rule_config
    content: "<describe-config-rules output>"
  - type: compliance_details
    content: "<compliance details for affected resources>"
  - type: resource_config
    content: "<resource configuration history>"
severity: MEDIUM
mitigation:
  immediate: "Correct rule parameters and force re-evaluation"
  long_term: "Document parameter requirements and set up compliance monitoring"
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
  - "NEVER suggest disabling Config rules to fix compliance"
  - "NEVER suggest deleting the configuration recorder"
  - "NEVER remove remediation actions without review"

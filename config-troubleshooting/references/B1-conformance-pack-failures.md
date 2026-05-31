---
title: "B1 — Conformance Pack Failures"
description: "Diagnose AWS Config conformance pack deployment and evaluation failures"
status: active
severity: HIGH
triggers:
  - "conformance pack failed"
  - "conformance pack error"
  - "pack deployment failure"
  - "conformance pack not deploying"
  - "pack creation failed"
owner: devops-agent
objective: "Identify and resolve conformance pack deployment failures and evaluation issues"
context: "Conformance packs are CloudFormation-based collections of Config rules. Failures can occur during deployment (CloudFormation stack errors), parameter resolution, rule limit violations, or IAM permission issues. The conformance pack status and CloudFormation stack events provide diagnostic details."
---

## Phase 1 — Triage

MUST:
- Check conformance pack status: `aws configservice describe-conformance-pack-status`
- Describe the conformance pack: `aws configservice describe-conformance-packs --conformance-pack-names <pack-name>`
- Check compliance by conformance pack: `aws configservice describe-conformance-pack-compliance --conformance-pack-name <pack-name>`
- Verify recorder is running: `aws configservice describe-configuration-recorder-status`
- Check the Config service role permissions: `aws iam get-role --role-name <config-role>`

SHOULD:
- Check CloudFormation stack events for the conformance pack stack: `aws cloudformation describe-stack-events --stack-name awsconfigconforms-<pack-name>`
- Verify the S3 bucket for conformance pack templates: `aws s3 ls s3://<template-bucket>/`
- Check CloudTrail for conformance pack API errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=PutConformancePack --max-results 10`

MAY:
- Validate the conformance pack template syntax
- Check for rule name conflicts with existing rules
- Verify parameter values match expected types

## Phase 2 — Remediate

MUST:
- Fix CloudFormation template errors identified in stack events
- Ensure the conformance pack has 50 or fewer rules
- Verify IAM permissions for the Config service role

SHOULD:
- Re-deploy the conformance pack after fixing issues
- Validate template parameters before deployment
- Check for naming conflicts with existing Config rules

MAY:
- Split large conformance packs into multiple smaller packs
- Use AWS sample conformance pack templates as a starting point

## Common Issues

- symptoms: "Conformance pack status shows CREATE_FAILED"
  diagnosis: "CloudFormation stack deployment failed due to template errors, parameter issues, or IAM permissions."
  resolution: "Check CloudFormation stack events for the specific error. Fix template and redeploy."

- symptoms: "Conformance pack exceeds maximum number of rules"
  diagnosis: "The pack contains more than 50 Config rules."
  resolution: "Split the conformance pack into multiple packs with 50 or fewer rules each."

- symptoms: "Conformance pack shows INSUFFICIENT_DATA for all rules"
  diagnosis: "The configuration recorder is not running or not recording the required resource types."
  resolution: "Start the recorder and ensure it covers the resource types evaluated by the pack's rules."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix CloudFormation template | YELLOW | Template fix; redeploy needed |
| Redeploy conformance pack | YELLOW | Deploys rules; can be deleted |
| Fix IAM permissions | YELLOW | Permission change; reversible |

## Escalation Conditions

- Recorder configuration changes
- Remediation action deployment
- Conformance pack deployment to production

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-conformance-pack-status` | LOW | Pack deployment status |
| `describe-conformance-pack-compliance` | LOW | Compliance summary |
| CloudFormation stack events | MEDIUM | Deployment details |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Conformance pack deployment | Delete via `delete-conformance-pack` |
| IAM permission change | Revert IAM policy |

## Output Format

```yaml
root_cause: "conformance_pack_failure — <specific_cause>"
evidence:
  - type: pack_status
    content: "<describe-conformance-pack-status output>"
  - type: cloudformation_events
    content: "<stack events showing failure reason>"
  - type: iam_role
    content: "<service role permissions>"
severity: HIGH
mitigation:
  immediate: "Fix the deployment issue and redeploy the conformance pack"
  long_term: "Validate templates before deployment and monitor pack status"
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

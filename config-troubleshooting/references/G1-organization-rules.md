---
title: "G1 — Organization Config Rules"
description: "Diagnose AWS Config organization rule deployment and evaluation issues"
status: active
severity: HIGH
triggers:
  - "organization rule failed"
  - "org rule not deploying"
  - "organization config rule"
  - "org rule error"
  - "member account rule"
owner: devops-agent
objective: "Identify and resolve organization Config rule deployment failures across member accounts"
context: "Organization Config rules deploy to all member accounts (or specified OUs). Failures can occur due to Organizations permissions, member account recorder issues, delegated admin configuration, or per-account rule limits. Only the management account or delegated admin can manage these rules."
---

## Phase 1 — Triage

MUST:
- List organization config rules: `aws configservice describe-organization-config-rules`
- Check organization rule status: `aws configservice describe-organization-config-rule-statuses`
- Get detailed status per account: `aws configservice get-organization-config-rule-detailed-status --organization-config-rule-name <rule-name>`
- Verify Organizations access: `aws organizations describe-organization`
- Check delegated admin: `aws organizations list-delegated-administrators --service-principal config-multiaccountsetup.amazonaws.com`

SHOULD:
- Verify the caller is management account or delegated admin
- Check if member accounts have Config recorder running
- Check per-account rule limits: `aws service-quotas get-service-quota --service-code config --quota-code L-8E14B865`

MAY:
- Check CloudTrail for organization rule events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=PutOrganizationConfigRule --max-results 10`
- Verify excluded accounts list if some accounts should be skipped

## Phase 2 — Remediate

MUST:
- Ensure the caller is management account or delegated admin
- Fix member account issues identified in detailed status
- Enable Config recorder in member accounts that are failing

SHOULD:
- Re-deploy the organization rule after fixing issues
- Verify Organizations trusted access is enabled for Config

MAY:
- Exclude specific accounts that cannot support the rule
- Set up monitoring for organization rule deployment status

## Common Issues

- symptoms: "Organization rule shows CREATE_FAILED for specific member accounts"
  diagnosis: "Those member accounts do not have Config recorder running or have reached the rule limit."
  resolution: "Enable Config recorder in failing accounts. Request rule limit increase if needed."

- symptoms: "AccessDeniedException when creating organization config rule"
  diagnosis: "The caller is not the management account or delegated admin."
  resolution: "Use the management account or register a delegated admin for Config."

- symptoms: "Organization rule exists but member accounts show no compliance data"
  diagnosis: "Member account recorders are not recording the resource types the rule evaluates."
  resolution: "Ensure member account recorders cover the required resource types."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix member account issues | YELLOW | Per-account changes; reversible |
| Enable Config in member accounts | YELLOW | Service enablement; reversible |
| Redeploy organization rule | YELLOW | Org-wide deployment; can be deleted |

## Escalation Conditions

- Recorder configuration changes
- Remediation action deployment
- Organization-wide rule changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-organization-config-rule-statuses` | LOW | Rule status |
| `get-organization-config-rule-detailed-status` | MEDIUM | Per-account status |
| `describe-organization` | MEDIUM | Organization details |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules without explicit approval

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Organization rule deployment | Delete via `delete-organization-config-rule` |
| Member account Config enablement | Stop recorder in member (not recommended) |

## Output Format

```yaml
root_cause: "organization_rule_failure — <specific_cause>"
evidence:
  - type: org_rule_status
    content: "<describe-organization-config-rule-statuses output>"
  - type: detailed_status
    content: "<per-account deployment status>"
  - type: organizations
    content: "<organization and delegated admin configuration>"
severity: HIGH
mitigation:
  immediate: "Fix member account issues and redeploy the organization rule"
  long_term: "Ensure all member accounts have Config properly configured"
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

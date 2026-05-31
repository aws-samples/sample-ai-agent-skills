---
title: "B2 — Organization Conformance Pack Issues"
description: "Diagnose organization-level conformance pack deployment and compliance issues"
status: active
severity: HIGH
triggers:
  - "organization conformance pack"
  - "org conformance pack failed"
  - "organization pack not deploying"
  - "member account conformance"
  - "org pack error"
owner: devops-agent
objective: "Identify and resolve organization conformance pack deployment failures across member accounts"
context: "Organization conformance packs deploy Config rules to all member accounts in an organization. Failures can occur due to Organizations permissions, member account recorder issues, delegated admin configuration, or per-account deployment failures. Status must be checked at both organization and member account levels."
---

## Phase 1 — Triage

MUST:
- Check organization conformance pack status: `aws configservice describe-organization-conformance-pack-statuses`
- Describe organization conformance packs: `aws configservice describe-organization-conformance-packs`
- Get detailed status per account: `aws configservice get-organization-conformance-pack-detailed-status --organization-conformance-pack-name <pack-name>`
- Verify Organizations access: `aws organizations describe-organization`
- Check delegated admin: `aws organizations list-delegated-administrators --service-principal config-multiaccountsetup.amazonaws.com`

SHOULD:
- Verify the management account or delegated admin is running the command
- Check if member accounts have Config recorder running
- Review CloudTrail for organization conformance pack events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=PutOrganizationConformancePack --max-results 10`

MAY:
- Check specific member account conformance pack status by assuming role into the member account
- Verify the S3 bucket for templates is accessible from all member accounts

## Phase 2 — Remediate

MUST:
- Ensure the caller is the management account or delegated admin
- Fix member account issues identified in detailed status
- Verify Organizations trusted access is enabled for Config

SHOULD:
- Enable Config recorder in member accounts that are failing
- Re-deploy the organization conformance pack after fixing issues

MAY:
- Exclude specific accounts or OUs if they cannot support the pack
- Set up monitoring for organization conformance pack deployment status

## Common Issues

- symptoms: "Organization conformance pack shows CREATE_FAILED for specific member accounts"
  diagnosis: "Those member accounts do not have Config recorder running or lack required permissions."
  resolution: "Enable Config recorder in the failing member accounts and ensure the Config service-linked role exists."

- symptoms: "AccessDeniedException when creating organization conformance pack"
  diagnosis: "The caller is not the management account or delegated admin, or Organizations trusted access is not enabled."
  resolution: "Use the management account or register a delegated admin. Enable trusted access for Config."

- symptoms: "Organization conformance pack deploys but shows no compliance data"
  diagnosis: "Member accounts have the rules but their recorders are not recording the required resource types."
  resolution: "Ensure member account recorders cover the resource types evaluated by the pack's rules."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix member account issues | YELLOW | Per-account changes; reversible |
| Enable Config in member accounts | YELLOW | Service enablement; reversible |
| Redeploy organization pack | YELLOW | Org-wide deployment; can be deleted |

## Escalation Conditions

- Recorder configuration changes
- Remediation action deployment
- Organization-wide conformance pack changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-organization-conformance-pack-statuses` | LOW | Pack status |
| `get-organization-conformance-pack-detailed-status` | MEDIUM | Per-account status |
| `describe-organization` | MEDIUM | Organization details |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Organization pack deployment | Delete via `delete-organization-conformance-pack` |
| Member account Config enablement | Stop recorder in member (not recommended) |

## Output Format

```yaml
root_cause: "organization_conformance_failure — <specific_cause>"
evidence:
  - type: org_pack_status
    content: "<describe-organization-conformance-pack-statuses output>"
  - type: detailed_status
    content: "<per-account deployment status>"
  - type: organizations
    content: "<organization and delegated admin configuration>"
severity: HIGH
mitigation:
  immediate: "Fix member account issues and redeploy the organization conformance pack"
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

---
title: "A4 — Region Configuration Issues"
description: "Diagnose issues with Control Tower governed region configuration"
status: active
severity: HIGH
triggers:
  - "Region not governed"
  - "Region deny guardrail blocking"
  - "Cannot deploy to region"
  - "Region decommission"
owner: devops-agent
objective: "Resolve region governance configuration issues"
context: "Control Tower governs specific regions. The region deny SCP blocks API calls in non-governed regions. Adding or removing governed regions requires landing zone update. Global services have exceptions in the region deny SCP."
---

## Phase 1 — Triage

MUST:
- Check governed regions: `aws controltower get-landing-zone --landing-zone-identifier <arn>` → manifest
- Check if region deny control is enabled: `aws controltower list-enabled-controls --target-identifier <ou-arn>`
- Identify which region is needed and whether it's currently governed
- Check if the issue is the region deny SCP blocking legitimate operations

SHOULD:
- Review the region deny SCP exceptions list for global services
- Check if the region is an opt-in region that needs to be enabled first
- Verify Config and CloudTrail are deployed in the target region

MAY:
- Check if StackSets have deployed successfully to the target region
- Review CloudTrail for denied API calls in the blocked region

## Phase 2 — Remediate

MUST:
- To add a governed region: update the landing zone configuration
- To work around region deny: verify the service is in the exceptions list
- For opt-in regions: enable the region in account settings first

SHOULD:
- Re-register OUs after adding a new governed region
- Verify all baseline StackSets deploy to the new region
- Test control enforcement in the new region

MAY:
- Request a region deny SCP exception for specific use cases (causes drift if done directly)

## Common Issues

- symptoms: "API calls blocked in a region the team needs"
  diagnosis: "Region deny SCP is blocking calls in a non-governed region."
  resolution: "Add the region as a governed region via landing zone update, or verify the service has a global exception."

- symptoms: "Config rules not evaluating in a specific region"
  diagnosis: "The region is not governed by Control Tower."
  resolution: "Add the region to governed regions and re-register OUs."

- symptoms: "Opt-in region not available"
  diagnosis: "The region must be enabled at the account level before CT can govern it."
  resolution: "Enable the opt-in region in account settings, then add to CT governed regions."

## Safety Ratings

```
safety_ratings:
  - "get-landing-zone: GREEN — read-only landing zone and region config"
  - "list-enabled-controls: GREEN — read-only control listing"
  - "describe-trails: GREEN — read-only CloudTrail config"
  - "enable opt-in region: YELLOW — recoverable, can be disabled again"
  - "update landing zone to add governed region: RED — deploys baselines to new region across all accounts, can break governance"
  - "modify region deny SCP exception: RED — changes security boundary for all governed accounts"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Region deny SCP needs exceptions for services not in the default list
- Adding a governed region affects production accounts across the organization

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries, region deny rules)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: Governed region list and configuration

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- If a governed region was added and causes issues, removing it requires another landing zone update — plan carefully
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- Document the current governed regions list before making changes
- If region deny SCP was modified directly (causing drift), use CT re-register OU to restore — do NOT manually edit the SCP

## Output Format

```yaml
root_cause: "region_configuration — <specific issue>"
evidence:
  - type: landing_zone_config
    content: "<governed regions list>"
  - type: scp_evaluation
    content: "<region deny SCP details>"
severity: HIGH
mitigation:
  immediate: "Add region to governance or verify SCP exceptions"
  long_term: "Document governed region strategy and review periodically"
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
  - "NEVER suggest disabling mandatory controls"
  - "NEVER suggest removing the audit account"
  - "NEVER modify landing zone without understanding blast radius"

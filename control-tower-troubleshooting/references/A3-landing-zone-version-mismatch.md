---
title: "A3 — Landing Zone Version Mismatch"
description: "Diagnose issues caused by running an outdated landing zone version"
status: active
severity: MEDIUM
triggers:
  - "Landing zone version outdated"
  - "New controls not available"
  - "Feature not supported in current version"
owner: devops-agent
objective: "Identify version-related limitations and plan an upgrade path"
context: "Control Tower releases new landing zone versions with additional controls, features, and bug fixes. Running an old version may lack features, have known issues, or miss security improvements."
---

## Phase 1 — Triage

MUST:
- Check current version: `aws controltower get-landing-zone --landing-zone-identifier <arn>` → version
- Check latest available version in CT documentation or console
- Identify which features or controls require a newer version
- Check if drift exists that would block an update

SHOULD:
- Review release notes for all versions between current and latest
- Identify any breaking changes in intermediate versions
- Check if any enabled controls are deprecated in newer versions

MAY:
- Compare control availability between current and target versions
- Assess impact of new mandatory controls in newer versions

## Phase 2 — Remediate

MUST:
- Plan the update path (some versions require sequential updates)
- Resolve drift before updating
- Review release notes for prerequisites

SHOULD:
- Test in a non-production landing zone if available
- Schedule update during low-activity period
- Notify account owners of the planned update

MAY:
- Enable new controls available in the updated version
- Re-register OUs to pick up new baseline configurations

## Common Issues

- symptoms: "Cannot enable a specific control, not available"
  diagnosis: "The control requires a newer landing zone version."
  resolution: "Update the landing zone to the version that supports the control."

- symptoms: "Console shows landing zone update available"
  diagnosis: "A newer version is available with improvements."
  resolution: "Review release notes and plan the update."

## Safety Ratings

```
safety_ratings:
  - "get-landing-zone: GREEN — read-only version check"
  - "list-enabled-controls: GREEN — read-only control listing"
  - "re-register OUs after update: YELLOW — recoverable, resets CT baselines"
  - "update landing zone to new version: RED — can break governance across all OUs and accounts"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Update path requires sequential version upgrades with breaking changes
- Deprecated controls in newer version are currently enforced in production
- New mandatory controls in target version conflict with existing workloads

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: Landing zone version and feature configuration

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- Document the current landing zone version and all enabled controls before starting the update
- If new mandatory controls break workloads after update, work with account owners to remediate — do NOT attempt to downgrade the landing zone

## Output Format

```yaml
root_cause: "version_mismatch — current <version> vs required <version>"
evidence:
  - type: landing_zone_config
    content: "<current version and target version>"
severity: MEDIUM
mitigation:
  immediate: "Plan and execute landing zone update"
  long_term: "Establish regular landing zone update cadence"
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

---
title: "Z1 — General Control Tower Troubleshooting (Catch-All)"
description: "Fallback SOP for Control Tower issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Control Tower issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Check landing zone status: `aws controltower get-landing-zone --landing-zone-identifier <arn>`
- Check for drift (landing zone, OU, account, SCP)
- Check recent CT operations: `aws controltower list-landing-zone-operations`
- Check Organizations structure: `aws organizations list-roots` → `list-organizational-units-for-parent`
- Check Service Catalog provisioned products for account issues
- Check IAM Identity Center status: `aws sso-admin list-instances`

SHOULD:
- Check CloudTrail for recent CT-related API calls
- Check StackSet health for AWSControlTowerBP-* StackSets
- Check Config compliance across accounts
- Review recent changes to SCPs, OUs, or accounts

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Landing zone not ACTIVE → Landing Zone (A1-A4)
- Account provisioning failed → Account Provisioning (B1-B4)
- Controls not working → Controls (C1-C4)
- Drift detected → Drift (D1-D4)
- OU issues → Organizational Units (E1-E3)
- SSO/access issues → IAM Identity Center (F1-F3)
- Logging broken → Logging & Monitoring (G1-G3)
- Customization failures → Customizations (H1-H3)
- StackSet problems → StackSet Operations (I1-I3)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all CT-managed resources for modifications
- Review CloudTrail for Organization-level changes
- Check all StackSets for failed instances
- Verify CT IAM roles exist in management and member accounts

SHOULD:
- Check AWS Health Dashboard for service events
- Compare current state with expected CT baseline
- Check if the issue correlates with a recent landing zone update

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Safety Ratings

```
safety_ratings:
  - "get-landing-zone: GREEN — read-only landing zone status"
  - "list-landing-zone-operations: GREEN — read-only operation listing"
  - "list-roots: GREEN — read-only Organizations root"
  - "list-organizational-units-for-parent: GREEN — read-only OU listing"
  - "list-instances (SSO): GREEN — read-only SSO instance listing"
  - "re-register OU: YELLOW — recoverable, resets CT baselines"
  - "repair landing zone: RED — modifies foundational governance structure"
  - "update landing zone: RED — can break governance across all OUs and accounts"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Fix requires moving the Audit or Log Archive account
- Drift affects multiple OUs or accounts
- Fix requires re-registering OUs in production
- Issue cannot be classified into any specific failure domain after investigation

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: StackSet health details (account IDs, regions, failure reasons)
- MEDIUM: SSO configuration and assignments

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- Document all investigation findings and evidence before attempting any remediation
- If the issue is unclassified, escalate rather than attempting broad remediation
- If multiple failure domains are affected, address them in order: landing zone → drift → OUs → accounts → controls

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<landing_zone|account|controls|drift|ou|sso|logging|customization|stackset|unknown>"
investigation_path: "landing zone status → drift check → operations → <domain_classification>"
evidence:
  - type: landing_zone_status
    content: "<landing zone details>"
  - type: organizations_state
    content: "<OU and account structure>"
  - type: stackset_health
    content: "<StackSet status>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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

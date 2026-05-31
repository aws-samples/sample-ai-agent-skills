---
title: "Z1 — General AWS Organizations Troubleshooting (Catch-All)"
description: "Fallback SOP for Organizations issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Organizations issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Describe the organization: `aws organizations describe-organization`
- List accounts: `aws organizations list-accounts --query 'Accounts[*].{Id:Id,Name:Name,Status:Status}'`
- List roots and policy types: `aws organizations list-roots`
- List SCPs: `aws organizations list-policies --filter SERVICE_CONTROL_POLICY`
- List OUs under root: `aws organizations list-organizational-units-for-parent --parent-id <root-id>`
- Check CloudTrail for recent Organizations events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=organizations.amazonaws.com --max-results 20`

SHOULD:
- List delegated administrators: `aws organizations list-delegated-administrators`
- List enabled service access: `aws organizations list-aws-service-access-for-organization`
- Check for failed account creations: `aws organizations list-create-account-status --states FAILED`
- Check for pending handshakes: `aws organizations list-handshakes-for-organization`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Account creation, closure, or suspension → Accounts (A1-A3)
- SCP evaluation or conflicts → SCPs (B1-B3)
- OU structure or limits → OUs (C1-C2)
- Delegated admin or trusted access → Delegation (D1-D2)
- Billing or cost allocation → Billing (E1-E2)
- Tag, AI opt-out, or backup policies → Policies (F1-F3)
- Service integration or features → Service Access (G1-G2)
- Account migration or restructuring → Migration (H1-H2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check the specific error message and CloudTrail event details
- Verify the caller is in the management account (most operations require it)
- Check for SCPs that may be blocking the operation
- Review organization feature set and enabled policy types

SHOULD:
- Check service quotas for Organizations limits
- Verify IAM permissions for the caller
- Review recent organizational changes

MAY:
- Check AWS Health Dashboard for Organizations service events
- Review AWS Support cases related to the organization

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<accounts|scps|ous|delegation|billing|policies|service_access|migration|unknown>"
investigation_path: "organization → accounts → SCPs → OUs → <domain_classification>"
evidence:
  - type: organization
    content: "<organization details>"
  - type: accounts
    content: "<account status summary>"
  - type: scps
    content: "<SCP summary>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings
- GREEN: describe-organization, list-accounts, list-policies
- YELLOW: attach-policy, create-organizational-unit, move-account
- RED: close-account, delete-policy, remove-account-from-organization

## Escalation Conditions
- "Remediation involves modifying SCPs in production"
- "Fix requires moving accounts between OUs"
- "Change affects the management account"
- "SCP modification could lock out member accounts"

## Rollback
- Pre-change: "Document current SCP attachments and OU structure"
- Verification: "Test affected account access after SCP change"
- Revert: "Detach modified SCP and reattach original"

## Data Sensitivity
- HIGH: "list-accounts reveals all account IDs, names, and emails"
- HIGH: "SCP content reveals security boundaries"
- MEDIUM: "OU structure reveals organizational hierarchy"

## Prohibited Actions
- NEVER suggest removing all SCPs from an OU
- NEVER suggest closing an account without confirming data backup
- NEVER suggest modifying the management account's policies
- NEVER suggest removing an account from the organization without root email access confirmation

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
  - command: "list-accounts"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-organization"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-policies"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest removing SCPs without understanding blast radius"
  - "NEVER suggest disabling AI opt-out policies without review"
  - "NEVER modify organization structure without approval"

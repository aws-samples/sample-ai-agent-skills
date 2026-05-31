---
title: "Z1 — General AWS Proton Troubleshooting (Catch-All)"
description: "Fallback SOP for Proton issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Proton issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- List all environments: `aws proton list-environments`
- List all services: `aws proton list-services`
- List environment templates: `aws proton list-environment-templates`
- List service templates: `aws proton list-service-templates`
- Check account settings: `aws proton get-account-settings`
- Check recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=proton.amazonaws.com --max-results 20`

SHOULD:
- Check CodeStar connections: `aws codestar-connections list-connections`
- List components: `aws proton list-components`
- Check for any resources in failed state

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Template registration/publishing issues → Templates (A1-A3)
- Environment provisioning/update failures → Environment (B1-B2)
- Service creation/pipeline failures → Service (C1-C2)
- Component provisioning/drift → Components (D1-D2)
- Template sync/Git integration → Sync (E1-E2)
- IAM role/CodeStar connection → Security (F1-F2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all resource deployment statuses systematically
- Review CloudTrail for recent configuration changes
- Verify IAM roles and permissions
- Check CloudFormation stacks for Proton-managed resources

SHOULD:
- Check AWS Health Dashboard for Proton service events
- Compare with known-good configurations
- Review template versions and their statuses

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<templates|environment|service|components|sync|security|unknown>"
investigation_path: "environments → services → templates → <domain_classification>"
evidence:
  - type: resource_status
    content: "<resource statuses summary>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — general troubleshooting uses read-only operations
- Service Disruption Risk: LOW — diagnostic commands do not affect deployments
- Compliance Impact: LOW — standard troubleshooting procedures

## Escalation Conditions
- Issue cannot be resolved using standard troubleshooting procedures
- Proton service becomes unavailable
- Multiple unrelated issues occur simultaneously
- AWS Health Dashboard shows Proton service event
- Issue requires AWS Support engagement

## Data Sensitivity
- Service configurations — INTERNAL
- CloudWatch metrics — INTERNAL
- Deployment logs — INTERNAL
- Error messages — INTERNAL
- Never expose deployment details in external communications

## Prohibited Actions
- Do NOT make changes to production deployments without approval
- Do NOT run destructive commands during initial diagnosis
- Do NOT share deployment details in public forums without sanitizing
- Do NOT bypass established escalation procedures
- Do NOT attempt fixes outside Proton service boundaries
- Do NOT ignore recurring issues

## Phase 3 — Rollback
- Revert any diagnostic changes made during troubleshooting
- Restore deployments to last known good configuration
- Document all troubleshooting steps and outcomes
- Verify deployment health after troubleshooting
- Update runbooks with new findings
- Close out temporary access granted for troubleshooting

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"

---
title: "D2 — Proton Drift Detection Issues"
description: "Diagnose and resolve infrastructure drift in Proton-managed resources"
status: active
severity: MEDIUM
triggers:
  - "drift detected"
  - "infrastructure drift"
  - "out of sync"
  - "desired state mismatch"
owner: devops-agent
objective: "Identify drift between Proton desired state and actual infrastructure, and resolve discrepancies"
context: "Proton drift detection identifies differences between the desired state (template) and actual infrastructure. Drift does NOT auto-remediate — manual intervention is required. Drift can be caused by manual changes, external tools, or failed updates."
---

## Phase 1 — Triage

MUST:
- Check environment deployment status for drift: `aws proton get-environment --name <env-name> --query '{Status:deploymentStatus,Message:deploymentStatusMessage}'`
- Check service instance for drift: `aws proton get-service-instance --name <instance-name> --service-name <service-name>`
- For CloudFormation, detect drift: `aws cloudformation detect-stack-drift --stack-name <stack-name>`
- Check drift results: `aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id <detection-id>`
- List drifted resources: `aws cloudformation describe-stack-resource-drifts --stack-name <stack-name> --stack-resource-drift-status-filters MODIFIED DELETED`

SHOULD:
- Identify who or what caused the drift (CloudTrail)
- Determine if drift is intentional or accidental
- Check if drift affects dependent services

MAY:
- Compare actual resource configuration with template expected values
- Check if drift was caused by auto-scaling or managed service updates

## Phase 2 — Remediate

MUST:
- Decide whether to accept drift or revert to desired state
- If reverting: update the environment/service to force redeployment
- If accepting: update the template to match the actual state

SHOULD:
- Document the drift cause and resolution
- Implement controls to prevent unauthorized manual changes
- Set up drift detection on a schedule

MAY:
- Use AWS Config rules to detect configuration changes
- Implement SCPs to prevent manual resource modification

## Common Issues

- symptoms: "Drift detected on environment resources"
  diagnosis: "Manual changes were made to Proton-managed resources outside of Proton."
  resolution: "Redeploy the environment to restore desired state, or update template to match."

- symptoms: "Service instance shows drift after environment update"
  diagnosis: "Environment output changes affected service instance resources."
  resolution: "Update the service instance to reconcile with new environment outputs."

- symptoms: "Drift on security group rules"
  diagnosis: "Manual security group modifications outside of Proton."
  resolution: "Redeploy to restore template-defined rules, or update template."

## Output Format

```yaml
root_cause: "infrastructure_drift — <specific_cause>"
evidence:
  - type: drift_status
    content: "<drifted resources and changes>"
  - type: cloudtrail
    content: "<who/what caused the drift>"
  - type: template_comparison
    content: "<desired vs actual state>"
severity: MEDIUM
mitigation:
  immediate: "Redeploy to restore desired state or update template"
  long_term: "Implement drift prevention controls and scheduled detection"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — drift detection is a monitoring activity
- Service Disruption Risk: MEDIUM — undetected drift can cause unexpected behavior
- Compliance Impact: HIGH — drift from approved configurations affects compliance

## Escalation Conditions
- Significant drift detected from approved configuration
- Drift causes service disruption
- Drift detection fails to identify changes
- Manual changes bypass Proton management
- Drift remediation fails

## Data Sensitivity
- Drift detection results — INTERNAL
- Configuration baselines — INTERNAL
- Resource state data — INTERNAL
- Remediation plans — INTERNAL
- Never expose drift details that reveal infrastructure vulnerabilities

## Prohibited Actions
- Do NOT ignore detected drift
- Do NOT make manual changes that bypass Proton management
- Do NOT disable drift detection
- Do NOT remediate drift without understanding the cause
- Do NOT bypass drift review processes
- Do NOT accept drift as normal without documentation

## Phase 3 — Rollback
- Remediate drift by re-applying Proton-managed configuration
- Restore resources to approved baseline
- Re-enable drift detection if disabled
- Verify configuration compliance after remediation
- Document drift causes and remediation steps
- Update change management procedures to prevent future drift

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

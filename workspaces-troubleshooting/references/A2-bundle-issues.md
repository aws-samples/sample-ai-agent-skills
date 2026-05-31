---
title: "A2 — Bundle Issues"
description: "Diagnose WorkSpace bundle creation, custom image, and compatibility problems"
status: active
severity: MEDIUM
triggers:
  - "bundle not found"
  - "custom bundle failed"
  - "workspace image error"
  - "bundle incompatible"
owner: devops-agent
objective: "Resolve WorkSpace bundle and custom image issues"
context: "Bundle issues arise from custom image creation failures, incompatible compute types, volume size constraints, or image validation errors. Custom bundles inherit volume sizes from the source WorkSpace and cannot be reduced."
---

## Phase 1 — Triage

MUST:
- List available bundles: `aws workspaces describe-workspace-bundles --owner AMAZON`
- Check custom bundle status: `aws workspaces describe-workspace-bundles --bundle-ids <bundle-id>`
- Verify custom image state: `aws workspaces describe-workspace-images --image-ids <image-id>`
- Check source WorkSpace state if creating custom bundle: `aws workspaces describe-workspaces --workspace-id <source-id>`

SHOULD:
- Verify bundle compute type is available in the target region
- Check if image creation is still in progress: `aws workspaces describe-workspace-images --image-ids <image-id> --query 'Images[].State'`
- Review CloudTrail for CreateWorkspaceBundle errors

MAY:
- Compare bundle configurations between regions
- Check if the source WorkSpace has pending Windows updates

## Phase 2 — Remediate

MUST:
- Wait for image creation to complete before creating bundle (can take 30-60 minutes)
- Ensure source WorkSpace is in AVAILABLE state before creating image
- Use compatible compute type for the bundle

SHOULD:
- Stop the source WorkSpace before creating an image for consistency
- Document volume sizes before creating custom bundles (cannot reduce later)

MAY:
- Create a new WorkSpace with desired volume sizes if current bundle volumes are too large

## Common Issues

- symptoms: "Custom image stuck in PENDING state"
  diagnosis: "Image creation takes 30-60 minutes. Source WorkSpace may have been in use during creation."
  resolution: "Wait for completion. If stuck beyond 2 hours, check source WorkSpace state and retry."

- symptoms: "Bundle not available in target region"
  diagnosis: "Not all Amazon bundles are available in every region. Custom bundles are region-specific."
  resolution: "Check available bundles in the target region. Recreate custom bundle in the needed region."

- symptoms: "Cannot reduce volume size in custom bundle"
  diagnosis: "Volume sizes are inherited from source WorkSpace and cannot be reduced."
  resolution: "Create a new WorkSpace with smaller volumes, then create a new custom bundle from it."

## Output Format

```yaml
root_cause: "bundle_issue — <specific_cause>"
evidence:
  - type: bundle_config
    content: "<bundle details and state>"
  - type: image_state
    content: "<custom image status>"
severity: MEDIUM
mitigation:
  immediate: "Fix bundle configuration or wait for image creation"
  long_term: "Standardize bundle creation process with documented volume sizes"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- WorkSpaces contain user desktop data and files
- Directory integration reveals Active Directory structure
- Connection logs contain user access patterns
- Classification: CONFIDENTIAL - user workspace data and directory information

## Prohibited Actions

- NEVER terminate or rebuild WorkSpaces without user data backup confirmation
- NEVER modify directory service configurations during business hours
- NEVER change MFA settings without user notification and transition period
- NEVER modify VPC or subnet configurations for running WorkSpaces without maintenance window

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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

---
title: "A3 — Proton Template Version Issues"
description: "Diagnose template version lifecycle and management problems"
status: active
severity: MEDIUM
triggers:
  - "version conflict"
  - "cannot publish version"
  - "version not found"
  - "major version"
  - "minor version"
owner: devops-agent
objective: "Identify and resolve template version lifecycle issues in Proton"
context: "Proton templates use semantic versioning with major and minor versions. Major version changes indicate breaking changes. Versions go through DRAFT → PUBLISHED → DEPRECATED lifecycle. Version management issues affect deployments and updates."
---

## Phase 1 — Triage

MUST:
- List all versions: `aws proton list-environment-template-versions --template-name <template-name>` or `aws proton list-service-template-versions --template-name <template-name>`
- Check specific version status: `aws proton get-environment-template-version --template-name <name> --major-version <major> --minor-version <minor>`
- Verify which version environments/services are using: `aws proton list-environments --query 'environments[*].{Name:name,TemplateMajor:templateMajorVersion,TemplateMinor:templateMinorVersion}'`

SHOULD:
- Check if recommended version is set: look at template's recommendedVersion field
- Verify no version conflicts between environments and services
- Check CloudTrail for version management events

MAY:
- Review version history for the template
- Check if deprecated versions are still in use

## Phase 2 — Remediate

MUST:
- Publish DRAFT versions before attempting deployment: `aws proton update-environment-template-version --template-name <name> --major-version <major> --minor-version <minor> --status PUBLISHED`
- Fix REGISTRATION_FAILED versions by correcting the bundle and re-registering
- Update environments/services to use the correct published version

SHOULD:
- Set recommended version for templates
- Deprecate old versions that should no longer be used
- Plan major version migrations with dependent service updates

MAY:
- Automate version management through template sync
- Create version compatibility matrices

## Common Issues

- symptoms: "Cannot deploy — template version not found"
  diagnosis: "Specified version doesn't exist or is not PUBLISHED."
  resolution: "Verify version exists and is in PUBLISHED status."

- symptoms: "Multiple DRAFT versions accumulating"
  diagnosis: "Versions registered but never published or cleaned up."
  resolution: "Publish needed versions and delete unnecessary DRAFT versions."

- symptoms: "Environment update fails after template version change"
  diagnosis: "New major version has breaking changes in inputs/outputs."
  resolution: "Review major version changes and update environment inputs accordingly."

## Output Format

```yaml
root_cause: "template_version_issue — <specific_cause>"
evidence:
  - type: version_status
    content: "<version lifecycle state>"
  - type: deployments
    content: "<environments/services using this version>"
severity: MEDIUM
mitigation:
  immediate: "Publish or fix the required version"
  long_term: "Implement version management policy and automation"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — version management is operational
- Service Disruption Risk: MEDIUM — version issues can affect deployments
- Compliance Impact: MEDIUM — version management affects configuration consistency

## Escalation Conditions
- Template version conflicts prevent deployments
- Version rollback fails
- Version compatibility issues between templates
- Version publishing fails
- Multiple version issues across templates

## Data Sensitivity
- Version configurations — INTERNAL
- Template version history — INTERNAL
- Compatibility matrices — INTERNAL
- Version metadata — INTERNAL
- Never expose version details that reveal infrastructure evolution

## Prohibited Actions
- Do NOT delete template versions used by active environments
- Do NOT publish untested template versions
- Do NOT bypass version validation
- Do NOT create incompatible version upgrades
- Do NOT ignore version deprecation notices
- Do NOT modify published versions without creating new versions

## Phase 3 — Rollback
- Revert to previous template versions
- Restore version configurations
- Verify deployment compatibility after version changes
- Document version issues and compatibility requirements
- Update version management procedures
- Restore version metadata if modified

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

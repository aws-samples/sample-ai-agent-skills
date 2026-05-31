---
title: "F2 — Exclusion Rules"
description: "Configure and troubleshoot Inspector scanning exclusion rules"
status: active
severity: LOW
triggers:
  - "exclude from scanning"
  - "exclusion rule"
  - "skip scanning"
  - "tag-based exclusion"
owner: devops-agent
objective: "Configure exclusion rules to control Inspector scanning scope"
context: "Inspector supports tag-based exclusion for EC2 instances and repository-based filtering for ECR. Exclusion rules help reduce noise from development/test resources and control scanning costs. Misconfigured exclusions can create unintended coverage gaps."
---

## Phase 1 — Triage

MUST:
- Check EC2 tag-based exclusion: `aws inspector2 batch-get-account-status --query 'Accounts[*].ResourceState'`
- Check ECR scanning rules: `aws ecr get-registry-scanning-configuration`
- Verify which resources are excluded: `aws inspector2 list-coverage --filter-criteria '{"scanStatusCode":[{"comparison":"EQUALS","value":"INACTIVE"}]}'`

SHOULD:
- Review tag values on excluded instances
- Check if exclusion rules are too broad
- Verify production resources are not accidentally excluded

MAY:
- Audit exclusion rules against resource inventory
- Compare covered vs total resources

## Phase 2 — Remediate

MUST:
- Review and correct exclusion tag keys and values
- Ensure production resources are not excluded
- Verify exclusion rules match intended scope

SHOULD:
- Use specific tags (e.g., InspectorExclude=true) rather than broad patterns
- Document exclusion policy and justification
- Regularly audit excluded resources

MAY:
- Create AWS Config rules to monitor exclusion compliance
- Set up alerts when production resources are tagged for exclusion

## Common Issues

- symptoms: "Production instances not being scanned"
  diagnosis: "Exclusion tag accidentally applied to production instances."
  resolution: "Remove exclusion tag from production instances."

- symptoms: "Exclusion rule not working"
  diagnosis: "Tag key or value doesn't match the exclusion configuration."
  resolution: "Verify exact tag key and value match the exclusion rule."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Review and correct exclusion tags | GREEN | Tag metadata changes |
| Remove exclusion tags from production | GREEN | Restores scanning coverage |
| Document exclusion policy | GREEN | Documentation only |

## Escalation Conditions

- Production resources accidentally excluded from scanning — escalate immediately to security team
- Exclusion rules too broad, hiding coverage gaps — escalate to security operations
- Exclusion policy changes — escalate to security team for review

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws inspector2 batch-get-account-status` | MEDIUM | Exposes scanning status |
| `aws inspector2 list-coverage` (INACTIVE) | MEDIUM | Exposes excluded resources |
| `aws ecr get-registry-scanning-configuration` | MEDIUM | Exposes scanning rules |

## Prohibited Actions

- **NEVER** suggest disabling Inspector scanning as a troubleshooting step
- **NEVER** suggest suppressing all findings without individual review
- **NEVER** exclude production or internet-facing resources from scanning
- **NEVER** create exclusion rules without documented justification

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Removed exclusion tags | Re-add exclusion tags if needed |
| Changed exclusion rules | Restore previous exclusion configuration |

## Output Format

```yaml
root_cause: "exclusion_rules — <specific_cause>"
evidence:
  - type: exclusion_config
    content: "<exclusion rule configuration>"
  - type: excluded_resources
    content: "<resources excluded from scanning>"
severity: LOW
mitigation:
  immediate: "Fix exclusion rules to match intended scope"
  long_term: "Establish exclusion governance and regular audits"
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
  - "NEVER suggest disabling scanning to reduce findings"
  - "NEVER suggest ignoring critical vulnerabilities"
  - "NEVER suggest removing SSM agent to avoid scans"

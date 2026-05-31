---
title: "B2 — False Positives"
description: "Identify and manage false positive findings in Inspector"
status: active
severity: LOW
triggers:
  - "false positive"
  - "incorrect finding"
  - "wrong vulnerability"
  - "not applicable"
owner: devops-agent
objective: "Identify false positive findings and manage them appropriately"
context: "False positives occur when Inspector reports a vulnerability that doesn't actually affect the resource. This can happen due to OS-level mitigations, custom builds, backported patches, or package detection inaccuracies. Proper handling involves verification before suppression."
---

## Phase 1 — Triage

MUST:
- Get finding details: `aws inspector2 get-finding --finding-arn <finding-arn>`
- Check the CVE details and affected package version
- Verify the package is actually installed on the resource
- Check if the OS vendor has backported the fix (common in RHEL, Amazon Linux)

SHOULD:
- Compare Inspector-reported version with actual installed version
- Check vendor security advisories for the specific CVE
- Verify if OS-level mitigations (SELinux, AppArmor) reduce the risk

MAY:
- Test the vulnerability to confirm it's not exploitable
- Check NVD for the CVE details and affected version ranges

## Phase 2 — Remediate

MUST:
- Verify the finding is truly a false positive before suppressing
- Document the reason for false positive determination
- Create a suppression rule if confirmed: `aws inspector2 create-filter --name <filter-name> --action SUPPRESS --filter-criteria '{"vulnerabilityId":[{"comparison":"EQUALS","value":"<CVE-ID>"}],"resourceId":[{"comparison":"EQUALS","value":"<resource-id>"}]}'`

SHOULD:
- Use specific suppression criteria (CVE + resource) rather than broad rules
- Review suppressed findings periodically
- Report confirmed false positives to AWS through support

MAY:
- Create organization-wide suppression rules for known false positives
- Track false positive rates over time

## Common Issues

- symptoms: "Finding for a CVE that was backport-patched"
  diagnosis: "Inspector detects package version but doesn't recognize vendor backport."
  resolution: "Verify with vendor advisory. Suppress with documented justification."

- symptoms: "Finding for a package not actually used"
  diagnosis: "Package installed as dependency but not actively used."
  resolution: "Remove unused package if possible. Suppress with justification if not."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create suppression rule for confirmed false positive | YELLOW | Hides findings from active view; must be justified |
| Document false positive determination | GREEN | Documentation only |
| Report to AWS Support | GREEN | No state change |

## Escalation Conditions

- High volume of false positives affecting triage efficiency — escalate to security operations
- Uncertainty about whether a finding is a true false positive — escalate to security team for verification
- False positive in compliance-critical system — escalate to compliance team before suppressing

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws inspector2 get-finding` | MEDIUM | Exposes CVE details and affected resource |
| Package version verification commands | LOW | System package information |

## Prohibited Actions

- **NEVER** suggest disabling Inspector scanning as a troubleshooting step
- **NEVER** suggest suppressing all findings without individual review
- **NEVER** suppress findings without documented justification
- **NEVER** create broad suppression rules that may hide real vulnerabilities

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created suppression rule | Delete the suppression rule: `aws inspector2 delete-filter --arn <filter-arn>` |
| Removed unused package | Reinstall the package if needed |

## Output Format

```yaml
root_cause: "false_positive — <specific_cause>"
evidence:
  - type: finding_details
    content: "<finding CVE and package info>"
  - type: verification
    content: "<verification that finding is false positive>"
severity: LOW
mitigation:
  immediate: "Suppress confirmed false positive with documentation"
  long_term: "Establish false positive review process"
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

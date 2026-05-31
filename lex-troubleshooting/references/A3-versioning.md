---
title: "A3 — Bot Versioning and Alias Issues"
description: "Diagnose bot version and alias management problems"
status: active
severity: MEDIUM
triggers:
  - "bot version error"
  - "alias not working"
  - "wrong bot version"
  - "version creation failed"
owner: devops-agent
objective: "Identify and resolve bot versioning and alias issues"
context: "Bot versions are immutable snapshots. Aliases point to versions. DRAFT is mutable for testing. Production should use numbered versions via aliases."
---

## Phase 1 — Triage

MUST:
- List versions: `aws lexv2-models list-bot-versions --bot-id <bot-id>`
- List aliases: `aws lexv2-models list-bot-aliases --bot-id <bot-id>`
- Check alias configuration: `aws lexv2-models describe-bot-alias --bot-id <bot-id> --bot-alias-id <alias-id>`
- Verify alias points to correct version

SHOULD:
- Check if all locales are built before version creation
- Verify alias routing configuration (traffic percentage)
- Check conversation logs configuration on alias

MAY:
- Review version creation history in CloudTrail
- Check for alias update failures

## Phase 2 — Remediate

MUST:
- Build all locales before creating a version
- Update alias to point to correct version
- Use numbered versions for production (not DRAFT)

SHOULD:
- Implement blue/green deployment with alias routing
- Tag versions for tracking

MAY:
- Automate version creation in CI/CD pipeline

## Common Issues

- symptoms: "Version creation fails"
  diagnosis: "One or more locales not built or build failed."
  resolution: "Build all locales successfully, then create version."

- symptoms: "Production bot behaves differently than test"
  diagnosis: "Alias points to old version, not latest."
  resolution: "Update alias to point to the new version."

## Output Format

```yaml
root_cause: "versioning — <specific_cause>"
evidence:
  - type: alias_config
    content: "<alias version mapping>"
  - type: version_list
    content: "<available versions>"
severity: MEDIUM
mitigation:
  immediate: "Update alias to correct version"
  long_term: "Implement version management automation"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete or overwrite a published bot alias that is actively serving production traffic
- **NEVER** modify intent or slot configurations on a live bot version without creating a new draft version first
- **NEVER** remove Lambda fulfillment hooks from intents without verifying downstream dependencies
- **NEVER** disable conversation logs on bots subject to compliance or audit requirements
- **NEVER** migrate a V1 bot to V2 in-place without maintaining the V1 bot as a fallback

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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

---
title: "Z1 — General Lake Formation Troubleshooting"
description: "Catch-all systematic investigation for Lake Formation issues"
status: active
severity: MEDIUM
triggers:
  - "Lake Formation issue"
  - "LF error"
  - "data lake problem"
  - "LF not working"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate Lake Formation issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for Lake Formation issues that don't clearly match a specific runbook. It covers LF settings, permissions, data locations, catalog resources, and systematic elimination of common causes."
---

## Phase 1 — Triage

MUST:
- Check LF data lake settings: `aws lakeformation get-data-lake-settings`
- List permissions for the affected principal: `aws lakeformation list-permissions --principal '{"DataLakePrincipalIdentifier":"<arn>"}'`
- List registered data locations: `aws lakeformation list-resources`
- Check IAMAllowedPrincipals status on affected resources

SHOULD:
- Check CloudTrail for recent LF events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=lakeformation.amazonaws.com --max-results 20`
- Verify Glue catalog resources exist
- Check for cross-account sharing configuration
- Review LF-Tag assignments

MAY:
- Check AWS Health Dashboard for LF service issues
- Review analytics service integration configuration

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Permission issues → A1 (grants), A2 (revokes), A3 (IAM vs LF)
  - Registration → B1 (data locations), B2 (S3 paths)
  - Catalog → C1 (db/table permissions), C2 (column security)
  - Cross-account → D1 (sharing), D2 (RAM integration)
  - Governed tables → E1 (transactions), E2 (compaction)
  - Blueprints → F1 (failures), F2 (workflows)
  - Integration → G1 (analytics services), G2 (Glue)
  - Security → H1 (data filters), H2 (LF-Tags)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Review recent changes that may have caused the issue

MAY:
- Set up comprehensive LF monitoring
- Create a post-incident report

## Common Issues

- symptoms: "Cannot access data through analytics service"
  diagnosis: "Multiple possible causes: LF permissions, IAM, IAMAllowedPrincipals, registration."
  resolution: "Check LF permissions → IAMAllowedPrincipals → data location registration → IAM."

- symptoms: "Permissions were working but stopped"
  diagnosis: "Recent permission change, tag change, or IAMAllowedPrincipals modification."
  resolution: "Check CloudTrail for recent changes. Verify current permission state."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: lf_settings
    content: "<LF data lake settings>"
  - type: permissions
    content: "<permission configuration>"
  - type: registrations
    content: "<data location registrations>"
severity: MEDIUM
general_analysis:
  lf_admins: "configured | not configured"
  iam_allowed_principals: "present | revoked"
  data_locations: "registered | not registered"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive LF monitoring and governance"
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

- **NEVER** revoke Lake Formation permissions on production databases without explicit customer approval
- **NEVER** modify IAMAllowedPrincipals settings without understanding the full permission inheritance chain
- **NEVER** deregister S3 data locations that are actively referenced by Glue tables or crawlers
- **NEVER** disable hybrid access mode on databases with active IAM-based consumers
- **NEVER** alter cross-account resource shares (RAM) without coordinating with all participating accounts

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
